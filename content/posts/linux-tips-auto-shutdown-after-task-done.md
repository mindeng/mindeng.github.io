
---
title: "Linux Tips: 任务完成后自动关机"
date: 2022-12-03T10:09:00.000Z
lastmod: 2022-12-04T13:09:00.000Z
tags: ['tools', 'linux']
draft: false
---


有时希望计算机在完成某项任务后自动关机，例如，正在使用 ``git`` 下载较大的代码仓库，又或者正用 ``wget`` 下载大文件，如果此时需要离开，又不想中断任务，最好的办法就是设置其完成后自动关机了。

这里介绍一种办法，利用 ``cron`` 机制监控任务的完成情况，并在任务结束后，执行关机指令。  
  
1.  获取任务的 pid :    
    
    ```bash
    ps -eo pid,command | grep git
    
    ```  
1.  编写关机脚本，假设存放在文件 ``/path/to/halt-if-gone`` 中（不要忘了加上可执行权限）:    
    
    ```bash
    #! /bin/bash
    
    if [[ $# -eq 1 ]]; then
      # 2分钟后关机，让你有反悔的余地
      ps -eo pid | grep $1 &> /dev/null || /sbin/shutdown -h +2
    fi
    
    ```  
1.  设置 ``cron`` 指令，每15分钟扫描一次（当然，你也可以缩短扫描间隔）::    
    
    ```bash
    # 请将 $TASK_PID 替换为你要等待的任务的 pid
    */15 * * * * /path/to/halt-if-gone $TASK_PID
    
    ```    
    