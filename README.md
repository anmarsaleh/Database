USE [SalaryLatDB]
GO
/****** Object:  Trigger [dbo].[SalaryRecords_Tracking]    Script Date: 6/4/2022 7:54:27 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER trigger [dbo].[SalaryRecords_Tracking] on [dbo].[SalaryRecords] for insert, update, delete
as

declare @bit int ,
@field int ,
@maxfield int ,
@char int ,
@fieldname varchar(128) ,
@TableName varchar(128) ,
@PKCols varchar(1000) ,
@sql varchar(2000), 
@UpdateDate varchar(21) ,
@UserName varchar(128) ,
@Type char(1) ,
@PKFieldSelect varchar(1000),
@PKValueSelect varchar(1000)
 
select @TableName = 'SalaryRecords'


select @UserName = system_user ,
@UpdateDate = convert(varchar(8), getdate(), 112) + ' ' + convert(varchar(12), getdate(), 114)


if exists (select * from inserted )
if exists (select * from deleted )
select @Type = 'U'
else
select @Type = 'I'
else
select @Type = 'D'


select * into #ins from inserted
select * into #del from deleted

select @UserName =  us.PcName from Users us Inner join inserted iu on iu.UserID=us.UserID
select @PKCols = coalesce(@PKCols + ' and', ' on') + ' i.' + c.COLUMN_NAME + ' = d.' + c.COLUMN_NAME
from INFORMATION_SCHEMA.TABLE_CONSTRAINTS pk ,
INFORMATION_SCHEMA.KEY_COLUMN_USAGE c
where pk.TABLE_NAME = @TableName
and CONSTRAINT_TYPE = 'PRIMARY KEY'
and c.TABLE_NAME = pk.TABLE_NAME
and c.CONSTRAINT_NAME = pk.CONSTRAINT_NAME
and c.COLUMN_NAME!='MarkRecord'

select @PKFieldSelect = coalesce(@PKFieldSelect+'+','') + '''' + COLUMN_NAME + '''' 
from INFORMATION_SCHEMA.TABLE_CONSTRAINTS pk ,
INFORMATION_SCHEMA.KEY_COLUMN_USAGE c
where pk.TABLE_NAME = @TableName
and CONSTRAINT_TYPE = 'PRIMARY KEY'
and c.TABLE_NAME = pk.TABLE_NAME
and c.CONSTRAINT_NAME = pk.CONSTRAINT_NAME
and c.COLUMN_NAME!='MarkRecord'

select @PKValueSelect = coalesce(@PKValueSelect+'+','') + 'convert(varchar(100), coalesce(i.' + COLUMN_NAME + ',d.' + COLUMN_NAME + '))'
from INFORMATION_SCHEMA.TABLE_CONSTRAINTS pk ,    
INFORMATION_SCHEMA.KEY_COLUMN_USAGE c   
where  pk.TABLE_NAME = @TableName   
and CONSTRAINT_TYPE = 'PRIMARY KEY'   
and c.TABLE_NAME = pk.TABLE_NAME   
and c.CONSTRAINT_NAME = pk.CONSTRAINT_NAME 
and c.COLUMN_NAME!='MarkRecord'

if @PKCols is null
begin
raiserror('no PK on table %s', 16, -1, @TableName)
return
end

select @field = 0, @maxfield = max(ORDINAL_POSITION) from INFORMATION_SCHEMA.COLUMNS where TABLE_NAME = @TableName and COLUMN_NAME!='MarkRecord'
while @field < @maxfield
begin
select @field = min(ORDINAL_POSITION) from INFORMATION_SCHEMA.COLUMNS where TABLE_NAME = @TableName and ORDINAL_POSITION > @field and COLUMN_NAME!='MarkRecord'
select @bit = (@field - 1 )% 8 + 1
select @bit = power(2,@bit - 1)
select @char = ((@field - 1) / 8) + 1
if substring(COLUMNS_UPDATED(),@char, 1) & @bit > 0 or @Type in ('I','D')
begin
select @fieldname = COLUMN_NAME from INFORMATION_SCHEMA.COLUMNS where TABLE_NAME = @TableName and ORDINAL_POSITION = @field and COLUMN_NAME!='MarkRecord'
select @sql = 'insert Audit (Type, TableName, PrimaryKeyField, PrimaryKeyValue, FieldName, OldValue, NewValue, UpdateDate, UserName,UserID,SalaryMonth,SalaryYear,EmpID)'
select @sql = @sql + ' select ''' + @Type + ''''
select @sql = @sql + ',''' + @TableName + ''''
select @sql = @sql + ',' + @PKFieldSelect
select @sql = @sql + ',' + @PKValueSelect
select @sql = @sql + ',''' + @fieldname + ''''
select @sql = @sql + ',convert(varchar(1000),d.' + @fieldname + ')'
select @sql = @sql + ',convert(varchar(1000),i.' + @fieldname + ')'
select @sql = @sql + ',''' + @UpdateDate + ''''
select @sql = @sql + ',''' + @UserName + ''''
select @sql = @sql + ',i.UserID'
select @sql = @sql + ',i.SalaryMount'
select @sql = @sql + ',i.SalaryYear'
select @sql = @sql + ',i.EmpID'
select @sql = @sql + ' from #ins i full outer join #del d'
select @sql = @sql + @PKCols
select @sql = @sql + ' where i.' + @fieldname + ' <> d.' + @fieldname 
select @sql = @sql + ' or (i.' + @fieldname + ' is null and  d.' + @fieldname + ' is not null)' 
select @sql = @sql + ' or (i.' + @fieldname + ' is not null and  d.' + @fieldname + ' is null)' 

exec (@sql)
end
end
