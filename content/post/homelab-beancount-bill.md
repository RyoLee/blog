---
author: Ryo
title: 通过Python自动生成Beancount月度账单
description: "通过BQL查询生成Latex格式Beancount月度账单, 再导出为PDF后发送到邮箱"
tags: ["Beancount", "BQL", "Python", "Latex", "记账"]
projects: ["Homelab"]
---

![截图](https://s1.ax1x.com/2023/02/03/pSsQws1.png)

### 背景

最近几个月开始使用 Beancount 记账, 并且每月通过 Github actions 自动生成一份 markdown 格式的汇总单([参考](https://www.bmpi.dev/self/beancount-my-accounting-tool-v2/#%E5%AE%9A%E6%97%B6%E8%8E%B7%E5%8F%96%E6%94%B6%E6%94%AF%E6%8A%A5%E8%A1%A8)), 由于觉得显示效果不太好, 决定手撸一份新的生成程序

### 代码

#### 文档生成

首先是生成 latex 文档的部分, 直接使用文本模板+占位符替换的模式实现, 这样就不用和 latex 的包打交道了(懒狗

```python
#!/usr/bin/env python3
# -*- coding: UTF-8 -*-

import datetime
from decimal import Decimal
from beancount import loader
from beancount.query import query
import calendar
import sys
import os
from dateutil.relativedelta import relativedelta
_template = r'''
\documentclass[UTF8]{ctexart}
\usepackage{multicol}
\usepackage{geometry}

\setlength\columnsep{1cm}
\geometry{a4paper,left=1.5cm,right=1.5cm,top=0cm,bottom=1.5cm}
\columnseprule=1pt

\title{Monthly Beancount Report}
\author{GitHub Actions}
\date{}

\begin{document}
\maketitle

\begin{center}\small
\textbf{\Large Statement of Comprehensive Income} \\
\date{ENV_YEAR-ENV_MONTH-01 to ENV_YEAR-ENV_MONTH-ENV_DAY}
\begin{multicols}{2}
\begin{center}
\textbf{Income Detail} \\
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
ACCOUNT & TOTAL(CNY) \\
\hline
ENV_LINES_INCOME
\end{tabular*}
\vfill\null
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Total & ENV_TOTAL_INCOME \\
\end{tabular*}
\end{center}
\columnbreak
\begin{center}
\textbf{Expenses Detail} \\
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
ACCOUNT & TOTAL(CNY) \\
\hline
ENV_LINES_EXPENSES
\end{tabular*}
\vfill\null
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Total & ENV_TOTAL_EXPENSES \\
\end{tabular*}
\end{center}
\end{multicols}
\begin{center}
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Net Profit & ENV_NET_PROFIT \\
\end{tabular*}
\end{center}
\vfill\null
\textbf{\Large Statement of Financial Position} \\
\date{As of ENV_YEAR-ENV_MONTH-ENV_DAY}
\begin{multicols}{2}
\begin{center}
\textbf{Assets Detail} \\
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
ACCOUNT & TOTAL(CNY) \\
\hline
ENV_LINES_ASSETS
\end{tabular*}
\vfill\null
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Total & ENV_TOTAL_ASSETS \\
\end{tabular*}
\end{center}
\columnbreak
\begin{center}
\textbf{Liabilities Detail} \\
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
ACCOUNT & TOTAL(CNY) \\
\hline
ENV_LINES_LIABILITIES
\end{tabular*}
\vfill\null
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Total & ENV_TOTAL_LIABILITIES \\
\end{tabular*}
\end{center}
\end{multicols}
\begin{center}
\begin{tabular*}{\hsize}{@{}@{\extracolsep{\fill}}lr@{}}
\hline
Debt Ratio & ENV_DEBT_RATIO \\
\end{tabular*}
\vfill\null
\end{center}
\end{center}
\end{document}
'''


class Buffer:
    def __init__(self, value: str):
        self.value = value

    def replace(self, _old: str, _new: str):
        self.value = self.value.replace(_old, _new)

    def __str__(self):
        return self.value


if __name__ == '__main__':
    # load file
    ledger_data, errors, options = loader.load_file(
        os.path.join(os.getcwd(), sys.argv[1]))

    # init buffer
    _buffer = Buffer(_template)

    # init time
    lm = datetime.datetime.now() - relativedelta(months=1)
    year = lm.strftime("%Y")
    month = lm.strftime("%m")
    ENV_YEAR = year
    ENV_MONTH = month
    year = int(year)
    month = int(month)
    day = calendar.monthrange(year, month)[1]
    ENV_DAY = str(day)

    # ENV_LINES_INCOME
    sql_query = r'SELECT account, abs(sum(cost(position))) as total WHERE account ~ "^Income:*" and year = ' + \
        ENV_YEAR + r' and month = ' + ENV_MONTH + \
        r' GROUP BY month, account ORDER BY account, total DESC'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_LINES_INCOME = ""
    for rrow in rrows:
        ENV_LINES_INCOME += rrow[0] + r' & ' + \
            '{0:.2f}'.format(rrow[1]) + r' \\' + '\n'

    # ENV_TOTAL_INCOME
    sql_query = r'SELECT abs(sum(cost(position))) WHERE account ~ "^Income:*" and year = ' + \
        ENV_YEAR + r' and month =  ' + ENV_MONTH + r' GROUP BY month'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_TOTAL_INCOME = '{0:.2f}'.format(rrows[0][0])

    # ENV_LINES_EXPENSES
    sql_query = r'SELECT account, sum(cost(position)) as total WHERE account ~ "^Expenses:*" and year = ' + \
        ENV_YEAR + r' and month = ' + ENV_MONTH + \
        r' GROUP BY month, account ORDER BY account, total DESC'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_LINES_EXPENSES = ""
    for rrow in rrows:
        ENV_LINES_EXPENSES += rrow[0] + r' & ' + \
            '{0:.2f}'.format(rrow[1]) + r' \\' + '\n'

    # ENV_TOTAL_EXPENSES
    sql_query = r'SELECT sum(cost(position)) WHERE account ~ "^Expenses:*" and year = ' + \
        ENV_YEAR + r' and month = ' + ENV_MONTH + r' GROUP BY month'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_TOTAL_EXPENSES = '{0:.2f}'.format(rrows[0][0])

    # ENV_NET_PROFIT
    sql_query = r'SELECT abs(sum(cost(position))) WHERE year = ' + ENV_YEAR + r' and month = ' + \
        ENV_MONTH + \
        r' and ( account ~ "^Expenses:*" or account ~ "^Income:*" ) GROUP BY month'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_NET_PROFIT = '{0:.2f}'.format(rrows[0][0])

    # ENV_LINES_ASSETS
    sql_query = r'SELECT account, sum(cost(position)) as total WHERE account ~ "^Assets:*" and year <= ' + \
        ENV_YEAR + r' and month <= ' + ENV_MONTH + r' and day <= ' + \
        ENV_DAY+r' ORDER BY account, total DESC'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_LINES_ASSETS = ""
    for rrow in rrows:
        ENV_LINES_ASSETS += rrow[0] + r' & ' + \
            '{0:.2f}'.format(rrow[1]) + r' \\' + '\n'

    # ENV_TOTAL_ASSETS
    sql_query = r'SELECT sum(cost(position)) as total WHERE account ~ "^Assets:*" and year <= ' + \
        ENV_YEAR + r' and month <= ' + ENV_MONTH + r' and day <= '+ENV_DAY
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_TOTAL_ASSETS = '{0:.2f}'.format(rrows[0][0])

    # ENV_LINES_LIABILITIES
    sql_query = r'SELECT account, abs(sum(cost(position))) as total WHERE account ~ "^Liabilities:*" and year <= ' + \
        ENV_YEAR + r' and month <= ' + ENV_MONTH + r' and day <= ' + \
        ENV_DAY+r' ORDER BY account, total DESC'
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_LINES_LIABILITIES = ""
    for rrow in rrows:
        ENV_LINES_LIABILITIES += rrow[0] + r' & ' + \
            '{0:.2f}'.format(rrow[1]) + r' \\' + '\n'

    # ENV_TOTAL_LIABILITIES
    sql_query = r'SELECT abs(sum(cost(position))) as total WHERE account ~ "^Liabilities:*" and year <= ' + \
        ENV_YEAR + r' and month <= ' + ENV_MONTH + r' and day <= '+ENV_DAY
    rrows = query.run_query(ledger_data, options, sql_query, numberify=True)[1]
    ENV_TOTAL_LIABILITIES = '{0:.2f}'.format(rrows[0][0])

    # ENV_DEBT_RATIO
    ENV_DEBT_RATIO = "{:.2%}".format(
        Decimal(ENV_TOTAL_LIABILITIES)/Decimal(ENV_TOTAL_ASSETS)).replace(r'%', r'\%')

    _buffer.replace(r"ENV_YEAR", ENV_YEAR)
    _buffer.replace(r"ENV_MONTH", ENV_MONTH)
    _buffer.replace(r"ENV_DAY", ENV_DAY)

    _buffer.replace(r"ENV_LINES_INCOME", ENV_LINES_INCOME)
    _buffer.replace(r"ENV_TOTAL_INCOME", ENV_TOTAL_INCOME)
    _buffer.replace(r"ENV_LINES_EXPENSES", ENV_LINES_EXPENSES)
    _buffer.replace(r"ENV_TOTAL_EXPENSES", ENV_TOTAL_EXPENSES)
    _buffer.replace(r"ENV_NET_PROFIT", ENV_NET_PROFIT)

    _buffer.replace(r"ENV_LINES_ASSETS", ENV_LINES_ASSETS)
    _buffer.replace(r"ENV_TOTAL_ASSETS", ENV_TOTAL_ASSETS)
    _buffer.replace(r"ENV_LINES_LIABILITIES", ENV_LINES_LIABILITIES)
    _buffer.replace(r"ENV_TOTAL_LIABILITIES", ENV_TOTAL_LIABILITIES)
    _buffer.replace(r"ENV_DEBT_RATIO", ENV_DEBT_RATIO)

    print(_buffer)
```

可以在这个[github 仓库](https://github.com/RyoLee/beancount-template/blob/master/tool/get_bill.py)查看后续更新

#### Github actions 配置

Github actions 配置部分比较简单, 每月月初定时生成后邮件发出即可

敏感信息已用\*号代替, 请自行替换(邮箱密码需在 github 添加 secrets `MAIL_PASSWORD`)

```yaml
name: schedule-email
on:
  schedule:
    - cron: "30 16 1 */1 *"
  workflow_dispatch:
jobs:
  make-monthly-bill:
    name: make-monthly-bill
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@master
      - name: Set up Python3
        uses: actions/setup-python@v1
        with:
          python-version: "3.7"
      - uses: BSFishy/pip-action@v1
        with:
          packages: |
            beancount
      - run: python3 tool/get_bill.py main.bean > report.tex
      - name: Github Action for LaTeX
        uses: xu-cheng/latex-action@v2
        with:
          root_file: report.tex
          latexmk_use_xelatex: true
      - name: send-email
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: ******
          server_port: ******
          username: ******
          password: ${{secrets.MAIL_PASSWORD}}
          subject: Monthly Beancount Report
          attachments: ./report.pdf
          to: ******
          from: ******
```
