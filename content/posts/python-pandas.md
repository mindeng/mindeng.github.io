
---
title: "Python Pandas"
date: 2022-09-05T07:00:00.000Z
lastmod: 2022-12-03T10:08:00.000Z
tags: ['tools', 'data-analysis', 'python']
draft: false
---



# 简单任务  
  
1.  数据读取    
    
    ```shell
    import pandas as pd
    
    df = pd.read_csv('data/android/oom-sessions/oom.csv')
    
    # thousands=',' 可以自动将类似 "1,316" 格式的数字转换为整数
    df = pd.read_csv('ui-90.csv', thousands=',')
    
    df.head()
    ```  
1.  数据过滤    
    
    ```shell
    df = df[df['name']=='ThreadPoolForeg']
    
    # 组合多个条件
    df = df[(df.p_date >= 20220830) & (df['count'] >= 200)]
    ```  
1.  数据排序    
    
    ```shell
    df = df.sort_values(by='size', ascending=False).reset_index(drop=True)
    ```  
1.  表合并    
    
    ```shell
    # 在多个列上做合并
    df = pd.merge(df1, df2, on=['p_date', 'AB实验分组名称'])
    ```  
1.  仅保留某些列、删除某些列    
    
    ```shell
    # 保留指定列
    df = df[['p_date', 'AB实验分组名称', '点击到用户可见首屏(90分位)', '点击到业务UI可见首屏(90分位)', '点击到用户可见首屏（均值）']]
    
    # 删除某些列
    ```  
1.  指定某列的取值，仅保留这些取值的行    
    
    ```shell
    # 所有 base 分组
    df_base = df.loc[df['AB实验分组名称'].isin(['base1', 'base2'])]
    
    # 所有实验分组
    df_exp = df.loc[df['AB实验分组名称'].isin(['exp1', 'exp2'])]
    ```  
1.  分别计算某些列的均值    
    
    ```shell
    # 分别计算这两列的均值
    df1[['duration1', 'duration2']].mean()
    ```  
1.  groupby, agg    
    
    ```python
    # 按 type 分组，并计算不同 type 的 counts 总和
    df = df.groupby(['type'], as_index=False)['counts'].agg(
                {'sum': 'sum'}
            )
    
    # 按 name 分组，并按分组分别计算 cost/上报数量 这两列的合
    df_names = df.groupby(['name'], as_index=False).agg({'cost': 'sum', '上报数量': 'sum'})
    ```  
1.  rename column name    
    
    ```python
    df.rename(columns={'oldName1': 'newName1', 'oldName2': 'newName2'}, inplace=True)
    ```  
1.  apply    
    
    ```python
    # 对列 a 应用一个 lambda 表达式
    df['a'] = df['a'].apply(lambda x: x + 1)
    ```    
    