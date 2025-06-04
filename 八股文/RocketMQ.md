url: /query_metadata
请求类型：POST
header: Content-Type: application/json
请求参数
字段名	字段类型	字段值示例	字段说明
table_name	string	"employee"	要查询的目标表名，必填。内部会转换为大写，与数据库中的表名匹配。
table_schema	string 或 null	"HR" 或 null	要查询的模式（Schema）名称，可选。如果不传或传 null，则使用代码中 DM8_CONFIG["schema"] 作为默认模式。

响应参数
字段名	字段类型	字段值示例	字段说明
fields	array of object	[<br> {"name":"EMP_ID","type":"NUMBER","comment":"员工编号"},<br> {"name":"EMP_NAME","type":"VARCHAR2","comment":"员工姓名"},<br> {"name":"DEPT_ID","type":"NUMBER","comment":""}<br>]	字段信息数组。每个元素为一个对象，包含该字段的名称、类型和注释
- name (string)：列名
- type (string)：数据类型
- comment (string)：列注释，如果无注释则为空字符串
primary_keys	array of string	["EMP_ID"]	主键列名称数组。如果该表有多个主键列，则依次列出；无主键时返回空数组。
table_comment	string	"员工表：存储公司员工基本信息"	表级注释，如果表没有注释则返回空字符串。
error	string（仅在出错时返回）	"ORA-00942: 表或视图不存在"	当查询过程中发生异常（如表不存在、连接失败等）时，会返回 HTTP 500 并附带此字段，表示具体的错误信息。

示例
请求示例

http
复制
编辑
POST /query_metadata
Content-Type: application/json

{
  "table_name": "employee",
  "table_schema": null
}
成功响应示例

json
复制
编辑
{
  "fields": [
    {"name":"EMP_ID","type":"NUMBER","comment":"员工编号"},
    {"name":"EMP_NAME","type":"VARCHAR2","comment":"员工姓名"},
    {"name":"DEPT_ID","type":"NUMBER","comment":""}
  ],
  "primary_keys": ["EMP_ID"],
  "table_comment": "员工表：存储公司员工基本信息"
}
错误响应示例（HTTP 500）

json
复制
编辑
{
  "error": "ORA-00942: 表或视图不存在"
}
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAzMzIwODc0LC0yMDg4NzQ2NjEyXX0=
-->