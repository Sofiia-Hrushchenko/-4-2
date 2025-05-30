-- Лабораторна робота 4: Створення функцій у MS SQL
-- Предметна область: Магазин корму для тварин "Dog&Cats"

-- Завдання 1-2
USE [Dogs&Cats]; 
GO

-- Завдання 3: Скалярний тип функцій
-- 3.1: Функція для отримання повного імені клієнта за його ID
IF OBJECT_ID('dbo.fnGetCustomerFullName', 'FN') IS NOT NULL
    DROP FUNCTION dbo.fnGetCustomerFullName;
GO
CREATE FUNCTION dbo.fnGetCustomerFullName (@CustomerID INT)
RETURNS NVARCHAR(101)
AS
BEGIN
    DECLARE @FullName NVARCHAR(101);
    SELECT @FullName = CONCAT(FirstName, ' ', LastName)
    FROM Customers
    WHERE CustomerID = @CustomerID;
    RETURN ISNULL(@FullName, 'N/A');
END;
GO
PRINT 'Функція dbo.fnGetCustomerFullName створена.';
GO

-- 3.2: Функція для розрахунку загальної суми конкретного замовлення
IF OBJECT_ID('dbo.fnGetOrderTotalAmount', 'FN') IS NOT NULL
    DROP FUNCTION dbo.fnGetOrderTotalAmount;
GO
CREATE FUNCTION dbo.fnGetOrderTotalAmount (@OrderID INT)
RETURNS DECIMAL(12, 2) -- точність для суми
AS
BEGIN
    DECLARE @Total DECIMAL(12, 2);
    SELECT @Total = SUM(Quantity * UnitPrice)
    FROM OrderDetails
    WHERE OrderID = @OrderID;
    RETURN ISNULL(@Total, 0.00);
END;
GO
PRINT 'Функція dbo.fnGetOrderTotalAmount створена.';
GO
-- Таблиця Orders вже має TotalAmount. Ця функція демонструє розрахунок.

-- 3.3: Функція для перевірки, чи є товар на складі (повертає кількість)
IF OBJECT_ID('dbo.fnGetProductStockQuantity', 'FN') IS NOT NULL
    DROP FUNCTION dbo.fnGetProductStockQuantity;
GO
CREATE FUNCTION dbo.fnGetProductStockQuantity (@ProductID INT)
RETURNS INT
AS
BEGIN
    DECLARE @Stock INT;
    SELECT @Stock = StockQuantity
    FROM Products
    WHERE ProductID = @ProductID;
    RETURN ISNULL(@Stock, 0);
END;
GO
PRINT 'Функція dbo.fnGetProductStockQuantity створена.';
GO

-- Завдання 4: Inline тип функцій 
-- (Inline Table-Valued Functions - ITVF)


-- 4.1: Функція, що повертає товари певної категорії
IF OBJECT_ID('dbo.fnGetProductsByCategory', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fnGetProductsByCategory;
GO
CREATE FUNCTION dbo.fnGetProductsByCategory (@CategoryID INT)
RETURNS TABLE
AS
RETURN
(
    SELECT ProductID, ProductName, Price, StockQuantity, AnimalType
    FROM Products
    WHERE CategoryID = @CategoryID
);
GO
PRINT 'Функція dbo.fnGetProductsByCategory створена.';
GO

-- 4.2: Функція, що повертає замовлення клієнта за певний період
IF OBJECT_ID('dbo.fnGetCustomerOrdersByDateRange', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fnGetCustomerOrdersByDateRange;
GO
CREATE FUNCTION dbo.fnGetCustomerOrdersByDateRange (@CustomerID INT, @StartDate DATETIME, @EndDate DATETIME)
RETURNS TABLE
AS
RETURN
(
    SELECT OrderID, OrderDate, TotalAmount, OrderStatus
    FROM Orders
    WHERE CustomerID = @CustomerID AND OrderDate >= @StartDate AND OrderDate <= @EndDate
);
GO
PRINT 'Функція dbo.fnGetCustomerOrdersByDateRange створена.';
GO

-- 4.3: Функція, що повертає співробітників на певній посаді
IF OBJECT_ID('dbo.fnGetEmployeesByPosition', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fnGetEmployeesByPosition;
GO
CREATE FUNCTION dbo.fnGetEmployeesByPosition (@Position NVARCHAR(50))
RETURNS TABLE
AS
RETURN
(
    SELECT EmployeeID, FirstName, LastName, HireDate, Salary
    FROM Employees
    WHERE Position = @Position
);
GO
PRINT 'Функція dbo.fnGetEmployeesByPosition створена.';
GO

-- Завдання 5: Multistate тип функцій 
-- (Multi-Statement Table-Valued Functions - MSTVF)

-- 5.1: Функція, що повертає топ N найдорожчих товарів з додаванням рангу
IF OBJECT_ID('dbo.fnGetTopNMostExpensiveProductsWithRank', 'TF') IS NOT NULL
    DROP FUNCTION dbo.fnGetTopNMostExpensiveProductsWithRank;
GO
CREATE FUNCTION dbo.fnGetTopNMostExpensiveProductsWithRank (@TopN INT)
RETURNS @TopProducts TABLE
(
    RankNo INT,
    ProductID INT,
    ProductName NVARCHAR(255),
    Price DECIMAL(10,2),
    CategoryName NVARCHAR(100)
)
AS
BEGIN
    INSERT INTO @TopProducts (RankNo, ProductID, ProductName, Price, CategoryName)
    SELECT
        ROW_NUMBER() OVER (ORDER BY p.Price DESC) as RankNo,
        p.ProductID,
        p.ProductName,
        p.Price,
        c.CategoryName
    FROM Products p
    JOIN Categories c ON p.CategoryID = c.CategoryID
    ORDER BY p.Price DESC
    OFFSET 0 ROWS FETCH NEXT @TopN ROWS ONLY;
    RETURN;
END;
GO
PRINT 'Функція dbo.fnGetTopNMostExpensiveProductsWithRank створена.';
GO

-- 5.2: Функція, що повертає історію замовлень клієнта з деталями (перші 2 товари кожного замовлення)
IF OBJECT_ID('dbo.fnGetCustomerOrderHistoryWithLimitedDetails', 'TF') IS NOT NULL
    DROP FUNCTION dbo.fnGetCustomerOrderHistoryWithLimitedDetails;
GO
CREATE FUNCTION dbo.fnGetCustomerOrderHistoryWithLimitedDetails (@CustomerID INT)
RETURNS @OrderHistory TABLE
(
    OrderID INT,
    OrderDate DATETIME,
    OrderStatus NVARCHAR(50),
    ProductName NVARCHAR(255),
    Quantity INT,
    UnitPrice DECIMAL(10,2)
)
AS
BEGIN
    INSERT INTO @OrderHistory (OrderID, OrderDate, OrderStatus, ProductName, Quantity, UnitPrice)
    SELECT
        o.OrderID,
        o.OrderDate,
        o.OrderStatus,
        p.ProductName,
        od.Quantity,
        od.UnitPrice
    FROM Orders o
    JOIN (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY OrderID ORDER BY ProductID) AS rn
        FROM OrderDetails
    ) od ON o.OrderID = od.OrderID AND od.rn <= 2 -- Обмеження на перші 2 товари
    JOIN Products p ON od.ProductID = p.ProductID
    WHERE o.CustomerID = @CustomerID
    ORDER BY o.OrderDate DESC, o.OrderID, od.rn;
    RETURN;
END;
GO
PRINT 'Функція dbo.fnGetCustomerOrderHistoryWithLimitedDetails створена.';
GO

-- 5.3: Функція, що повертає список товарів та їхню популярність (кількість разів у замовленнях)
IF OBJECT_ID('dbo.fnGetProductPopularity', 'TF') IS NOT NULL
    DROP FUNCTION dbo.fnGetProductPopularity;
GO
CREATE FUNCTION dbo.fnGetProductPopularity ()
RETURNS @ProductPopularity TABLE
(
    ProductID INT,
    ProductName NVARCHAR(255),
    TimesOrdered INT,
    TotalQuantitySold INT
)
AS
BEGIN
    INSERT INTO @ProductPopularity (ProductID, ProductName, TimesOrdered, TotalQuantitySold)
    SELECT
        p.ProductID,
        p.ProductName,
        COUNT(DISTINCT od.OrderID) AS TimesOrdered,
        SUM(ISNULL(od.Quantity,0)) AS TotalQuantitySold
    FROM Products p
    LEFT JOIN OrderDetails od ON p.ProductID = od.ProductID
    GROUP BY p.ProductID, p.ProductName
    ORDER BY TimesOrdered DESC, TotalQuantitySold DESC;
    RETURN;
END;
GO
PRINT 'Функція dbo.fnGetProductPopularity створена.';
GO


-- Завдання 6: Використовуючи існуючі типи виконати запити 
-- функції, які інкапсулюють логіку запитів,
-- подібних до тих, що були в попередніх лабораторних або практичних.

-- 6.1 (Скалярна): Функція для отримання середньої ціни товарів бренду
IF OBJECT_ID('dbo.fnGetAveragePriceByBrand', 'FN') IS NOT NULL
    DROP FUNCTION dbo.fnGetAveragePriceByBrand;
GO
CREATE FUNCTION dbo.fnGetAveragePriceByBrand (@BrandName NVARCHAR(100))
RETURNS DECIMAL(10,2)
AS
BEGIN
    DECLARE @AvgPrice DECIMAL(10,2);
    SELECT @AvgPrice = AVG(p.Price)
    FROM Products p
    JOIN Brands b ON p.BrandID = b.BrandID
    WHERE b.BrandName = @BrandName;
    RETURN ISNULL(@AvgPrice, 0.00);
END;
GO
PRINT 'Функція dbo.fnGetAveragePriceByBrand створена.';
GO

-- 6.2 (Inline TVF): Функція для пошуку товарів за частиною назви (LIKE)
IF OBJECT_ID('dbo.fnSearchProductsByName', 'IF') IS NOT NULL
    DROP FUNCTION dbo.fnSearchProductsByName;
GO
CREATE FUNCTION dbo.fnSearchProductsByName (@SearchTerm NVARCHAR(100))
RETURNS TABLE
AS
RETURN
(
    SELECT ProductID, ProductName, Price, StockQuantity
    FROM Products
    WHERE ProductName LIKE '%' + @SearchTerm + '%'
);
GO
PRINT 'Функція dbo.fnSearchProductsByName створена.';
GO

-- 6.3 (Multi-Statement TVF): Функція для отримання звіту по продажах за місяць
IF OBJECT_ID('dbo.fnGetMonthlySalesReport', 'TF') IS NOT NULL
    DROP FUNCTION dbo.fnGetMonthlySalesReport;
GO
CREATE FUNCTION dbo.fnGetMonthlySalesReport (@Year INT, @Month INT)
RETURNS @MonthlySales TABLE
(
    CategoryName NVARCHAR(100),
    TotalQuantitySold INT,
    TotalAmountSold DECIMAL(12,2)
)
AS
BEGIN
    INSERT INTO @MonthlySales (CategoryName, TotalQuantitySold, TotalAmountSold)
    SELECT
        c.CategoryName,
        SUM(od.Quantity) AS TotalQuantitySold,
        SUM(od.Quantity * od.UnitPrice) AS TotalAmountSold
    FROM Categories c
    JOIN Products p ON c.CategoryID = p.CategoryID
    JOIN OrderDetails od ON p.ProductID = od.ProductID
    JOIN Orders o ON od.OrderID = o.OrderID
    WHERE YEAR(o.OrderDate) = @Year AND MONTH(o.OrderDate) = @Month
    GROUP BY c.CategoryName
    ORDER BY TotalAmountSold DESC;
    RETURN;
END;
GO
PRINT 'Функція dbo.fnGetMonthlySalesReport створена.';
GO

-- Тестування функцій

-- Тестування скалярних функцій (Завдання 3)
PRINT 'Тестування скалярних функцій:';
SELECT dbo.fnGetCustomerFullName(1) AS Customer1_FullName;
SELECT dbo.fnGetOrderTotalAmount(1) AS Order1_TotalCalculated; 
SELECT dbo.fnGetProductStockQuantity(1) AS Product1_Stock;
GO

-- Тестування Inline Table-Valued функцій (Завдання 4)
PRINT 'Тестування Inline TVF:';
SELECT * FROM dbo.fnGetProductsByCategory(1); 
SELECT * FROM dbo.fnGetCustomerOrdersByDateRange(1, '2024-05-01', '2024-05-15');
SELECT * FROM dbo.fnGetEmployeesByPosition('Продавець-консультант');
GO

-- Тестування Multi-Statement Table-Valued функцій (Завдання 5)
PRINT 'Тестування Multi-Statement TVF:';
SELECT * FROM dbo.fnGetTopNMostExpensiveProductsWithRank(5);
SELECT * FROM dbo.fnGetCustomerOrderHistoryWithLimitedDetails(1); 
SELECT * FROM dbo.fnGetProductPopularity() ORDER BY TimesOrdered DESC, TotalQuantitySold DESC OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
GO

-- Тестування функцій з Завдання 6
PRINT 'Тестування функцій з Завдання 6:';
SELECT dbo.fnGetAveragePriceByBrand('Royal Canin') AS AvgPriceRoyalCanin;
SELECT * FROM dbo.fnSearchProductsByName('Adult');
SELECT * FROM dbo.fnGetMonthlySalesReport(2024, 5); 
GO

-- Завдання 7 - Підготовка запитів для аналізу продуктивності

PRINT 'ЗАПИТ 1 для аналізу: Виклик Multi-Statement TVF - fnGetProductPopularity';
PRINT 'Ця функція виконує агрегацію та LEFT JOIN.';
PRINT 'Команда для виконання в SSMS з увімкненим планом:';
PRINT 'SELECT * FROM dbo.fnGetProductPopularity() ORDER BY TimesOrdered DESC;';
SELECT * FROM dbo.fnGetProductPopularity() ORDER BY TimesOrdered DESC; 
GO

PRINT 'ЗАПИТ 2 для аналізу: Виклик Multi-Statement TVF - fnGetMonthlySalesReport';
PRINT 'Ця функція виконує кілька JOIN та агрегацію.';
PRINT 'Команда для виконання в SSMS з увімкненим планом:';
PRINT 'SELECT * FROM dbo.fnGetMonthlySalesReport(2024, 5);';
 SELECT * FROM dbo.fnGetMonthlySalesReport(2024, 5); 
GO


PRINT 'ЗАПИТ 3 для аналізу: Складний запит з попередніх лабораторних (наприклад, ЛР2, Завдання 11.5)';
PRINT 'Знайти для кожного бренду найпопулярніший товар (за кількістю проданих одиниць).';
PRINT 'Команда для виконання в SSMS з увімкненим планом:';

WITH ProductSales AS (
    SELECT
        p.BrandID,
        p.ProductID,
        p.ProductName,
        SUM(od.Quantity) AS TotalQuantitySold
    FROM Products p
    JOIN OrderDetails od ON p.ProductID = od.ProductID
    GROUP BY p.BrandID, p.ProductID, p.ProductName
),
RankedProductSales AS (
    SELECT
        ps.BrandID,
        ps.ProductID,
        ps.ProductName,
        ps.TotalQuantitySold,
        ROW_NUMBER() OVER (PARTITION BY ps.BrandID ORDER BY ps.TotalQuantitySold DESC) as rn
    FROM ProductSales ps
)
SELECT b.BrandName, rps.ProductName AS MostPopularProduct, rps.TotalQuantitySold
FROM Brands b
JOIN RankedProductSales rps ON b.BrandID = rps.BrandID
WHERE rps.rn = 1
ORDER BY b.BrandName;

PRINT 'SELECT b.BrandName, rps.ProductName AS MostPopularProduct, rps.TotalQuantitySold FROM Brands b JOIN (SELECT ps.BrandID, ps.ProductID, ps.ProductName, ps.TotalQuantitySold, ROW_NUMBER() OVER (PARTITION BY ps.BrandID ORDER BY ps.TotalQuantitySold DESC) as rn FROM (SELECT p.BrandID, p.ProductID, p.ProductName, SUM(od.Quantity) AS TotalQuantitySold FROM Products p JOIN OrderDetails od ON p.ProductID = od.ProductID GROUP BY p.BrandID, p.ProductID, p.ProductName) ps) rps ON b.BrandID = rps.BrandID WHERE rps.rn = 1 ORDER BY b.BrandName;';
GO

PRINT 'ЗАПИТ 4 для аналізу: Запит з використанням Inline TVF та додаткових умов';
PRINT 'Товари категорії "Сухий корм для собак" (CategoryID=1), дорожчі за 1000.';
PRINT 'Команда для виконання в SSMS з увімкненим планом:';
PRINT 'SELECT * FROM dbo.fnGetProductsByCategory(1) WHERE Price > 1000 ORDER BY Price DESC;';
SELECT * FROM dbo.fnGetProductsByCategory(1) WHERE Price > 1000 ORDER BY Price DESC;
GO


-- Приклади створення індексів 
-- Ці індекси можуть вже існувати, якщо стовпці є частиною PK або FK. 

-- Для таблиці Products
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_Products_CategoryID_Price' AND object_id = OBJECT_ID('dbo.Products'))
    CREATE NONCLUSTERED INDEX IX_Products_CategoryID_Price ON dbo.Products(CategoryID, Price) INCLUDE (ProductName, StockQuantity, AnimalType);
GO
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_Products_BrandID_Price' AND object_id = OBJECT_ID('dbo.Products'))
    CREATE NONCLUSTERED INDEX IX_Products_BrandID_Price ON dbo.Products(BrandID, Price) INCLUDE (ProductName);
GO
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_Products_ProductName' AND object_id = OBJECT_ID('dbo.Products'))
    CREATE NONCLUSTERED INDEX IX_Products_ProductName ON dbo.Products(ProductName) INCLUDE (Price, StockQuantity);
GO

-- Для таблиці Orders
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_Orders_OrderDate' AND object_id = OBJECT_ID('dbo.Orders'))
    CREATE NONCLUSTERED INDEX IX_Orders_OrderDate ON dbo.Orders(OrderDate) INCLUDE (CustomerID, EmployeeID, TotalAmount, OrderStatus);
GO
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_Orders_CustomerID' AND object_id = OBJECT_ID('dbo.Orders'))
    CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON dbo.Orders(CustomerID); 
GO

-- Для таблиці OrderDetails
IF NOT EXISTS (SELECT * FROM sys.indexes WHERE name = 'IX_OrderDetails_ProductID' AND object_id = OBJECT_ID('dbo.OrderDetails'))
    CREATE NONCLUSTERED INDEX IX_OrderDetails_ProductID ON dbo.OrderDetails(ProductID) INCLUDE (OrderID, Quantity, UnitPrice); 
GO
