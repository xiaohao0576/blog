---
title: "Hr Salary Working Days"
date: 2020-04-10T14:25:42+07:00
draft: false
---
## Salary rule中计算员工每天的薪资

### 问题描述
公司计算员工无薪假应扣除的薪资时，需要先计算每天的薪资，大概有以下几种算法:
1. 员工的工资 / 当月天数 （按日历表上计算）
2. 员工的工资 / 当月上班的天数 （去除周末，但包含公共节假日）
3. 员工的工资 / 当月实际应该出勤的天数 （去除周末和节假日）
但无论哪种算法，在salary rule中的python code中都无法直接获取当月应该上班的天数或小时数。

### 解决思路
1. 二次开发
2. 快速粗暴的方法

#### 先说二次开发的方法

salary rule中的python code部分，能够引用的对象是有限制的，这是因为代码是在Odoo的safe_eval方法中运行的，只看方法名就应该能想到会有很多限制。如果想深入了解代码运行原理，可以去查看hr.payslip的代码，里面有一个_get_payslip_lines()方法，方法中的localdict字典包含了在salary rule中可以引用的对象。

由于本人刚接触Odoo不久，不知道如何二次开发，所以这个方法留以后研究。

#### 再说快速粗暴的方法
直接离线算出每月应该工作的天数和小时数，然后硬编码到salary rule中。此方法适用的条件如下：

- 全公司只遵循一种上班时间制度，如果不同的员工有不同的上班时间安排，则不太适用此方法
- 公司不会频繁改变上班时间安排

### 上代码
```python
from pytz import timezone
from datetime import datetime


def get_work_days_data(employee, years=(2020, 2021, 2022)):
    """
    获取指定员工的工作日期和时间,如果需要获取满勤的工作天数和时长
    Args:
        employee: hr.employee对象
        years:获取哪几年的工作天数,元组或列表
    """
    tz=timezone(employee.tz)
    working_days = {}
    for year in years:
        month_data = []
        for month in range(1, 13):
            date_from = datetime(year, month, 1)
            if month < 12:
                date_to = datetime(year, month+1, 1)
            else:
                date_to = datetime(year+1, 1, 1)
            data = employee.get_work_days_data(from_datetime=tz.localize(
                date_from), to_datetime=tz.localize(date_to), compute_leaves=False)
            month_data.append(data)
        working_days[year]=month_data
    return working_days
```

以上代码保存到一个文件，比如workingdays.py
进入odoo-bin shell，然后import以上文件，并执行方法

``` python
./odoo-bin shell -d erp
import workingdays
user=self.env['hr.employee'].browse(2) # uid自己选取
work_data=workingdays.get_work_days_data(user,years=(2020,2021,2022))
print(work_data)

输出以下数据：
{2020: [{'days': 27.0, 'hours': 188.5}, {'days': 25.0, 'hours': 170.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}, {'days': 25.0, 'hours': 173.5}, {'days': 27.0, 'hours': 188.5}], 2021: [{'days': 26.0, 'hours': 177.5}, {'days': 24.0, 'hours': 166.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 188.5}], 2022: [{'days': 26.0, 'hours': 177.5}, {'days': 24.0, 'hours': 166.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}]}
```

在Slary rules中新建一个rule，计算方式选择Python code，输入以下代码可计算出员工的日薪,用于在无薪假期扣除规则中引用。
``` Python
work_data={2020: [{'days': 27.0, 'hours': 188.5}, {'days': 25.0, 'hours': 170.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}, {'days': 25.0, 'hours': 173.5}, {'days': 27.0, 'hours': 188.5}], 2021: [{'days': 26.0, 'hours': 177.5}, {'days': 24.0, 'hours': 166.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 188.5}], 2022: [{'days': 26.0, 'hours': 177.5}, {'days': 24.0, 'hours': 166.0}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 27.0, 'hours': 188.5}, {'days': 26.0, 'hours': 181.0}, {'days': 26.0, 'hours': 177.5}, {'days': 26.0, 'hours': 181.0}, {'days': 27.0, 'hours': 185.0}]}

year=payslip.date_from.year
month=payslip.date_from.month
month_working_days=work_data[year][month-1]['days']
result = categories.BASIC / month_working_days
```
