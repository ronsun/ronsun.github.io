---
title: 新增一個使用者定義函數提供亂數字串
date: 2017-08-20 20:41:01
categories:
- MSSQL

---

首先, 需求上是這樣的:  
要在資料庫中建立一個UDF(User-defined function), 它的功能是要能產生一組格式像這樣 zAc-jVu-euO-nQ7 的亂數字串。  

<!--more-->

參考許多網路文章並實作後, 發現在Function裡面並不能使用RAND()這類的函數, 所以只能先建立一個檢視表(View), 接著才能在函數中撈出產生在檢視表中的亂數來做動作。 

**建立檢視表**
``` sql
CREATE VIEW Get_RAND
AS
SELECT RAND() AS RandomNumber
GO
```

**建立函數**

``` sql
CREATE FUNCTION [dbo].[fu_CZ_NewID]()
RETURNS VARCHAR(100)
AS
BEGIN 
	--需求: 
	--  產生一個格式為 6ev-zS5-lMN-pwg 的亂數字串
	--  * 長度為15個字元
	--  * 每三個字元以'-'符號作為分隔符號

	--@id: 要產生的字串
	DECLARE @id VARCHAR(100) = '';

	--@length: 要產生的字串長度
	DECLARE @length INT = 15;  

	--@group: 每隔幾個字要加上分隔字元
	DECLARE @group INT = 3;

	---@chars: 亂數資料來源
	DECLARE @chars VARCHAR(100) = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz1234567890';

	--@count: 迴圈起始值
	DECLARE @count INT = 0;

	WHILE (@count < @length)
	BEGIN
		DECLARE @current INT = (SELECT RandomNumber FROM dbo.Get_RAND) * 100
		IF(@current <= LEN(@chars))
		BEGIN
			--判斷是否加分隔字元
			IF((LEN(@id) + 1) % (@group + 1) = 0)
			BEGIN
			SET @id += '-';
			SET @count += 1;
			END

			SET @id += SUBSTRING(@chars, @current, 1);
			SET @count += 1;
		END
	END

	-- Return the result of the function
	RETURN @id

END
```
