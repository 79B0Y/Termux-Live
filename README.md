在Root的安卓设备上，Termux中的进程被杀掉通常是由于安卓系统的内存管理（OOM Killer）、电池优化策略（Doze模式、App Standby）或者某些厂商定制的“省电精灵”、“后台管理”等功能导致的。

既然你有Root权限，那么可以采取更强力一些的措施。以下是一些方法，建议从上往下尝试：

**1. Termux 内部方法 (基础)**

*   **`termux-wake-lock`**:
    在Termux中执行耗时任务前，运行 `termux-wake-lock`。这会阻止CPU进入深度睡眠，并可能保持Wi-Fi连接。任务完成后，运行 `termux-wake-unlock` 释放锁。
    ```bash
    termux-wake-lock
    # 在这里运行你的长时间任务，例如：
    # python my_script.py &
    # ssh -R 8080:localhost:3000 user@remote_server
    # ...
    # 当你不再需要时，可以手动运行 termux-wake-unlock，或者在新会话中运行
    ```
    注意：这主要防止CPU休眠，但系统仍可能因内存不足而杀死进程。

*   **使用 `tmux` 或 `screen`**:
    这些终端复用器允许你创建持久化的会话。即使你关闭Termux应用或SSH断开，会话和其中的进程也会继续在后台运行（只要Termux主进程没被杀）。
    ```bash
    pkg install tmux # 或者 screen
    tmux new -s mysession
    # 在 tmux 会话中运行你的程序
    # 按 Ctrl+b 然后按 d 来分离会话 (detach)
    # 之后可以用 tmux attach -t mysession 重新连接
    ```

*   **`nohup ... &`**:
    让命令在后台运行，并且不受挂断信号(SIGHUP)影响，通常输出会重定向到 `nohup.out`。
    ```bash
    nohup ./my_long_running_script.sh &
    ```

**2. 安卓系统层面设置 (无需Root，但Root后更可靠)**

*   **电池优化白名单**:
    *   进入安卓系统的 设置 -> 应用 -> Termux -> 电池 (或 电池优化)。
    *   将Termux设置为“不受限制”或“不优化”。不同品牌手机的叫法可能不同。
    *   **Root后** 可以用命令尝试（不一定所有系统都有效）：
        ```bash
        su
        dumpsys deviceidle whitelist +com.termux
        ```

*   **保持Termux前台服务通知**:
    Termux默认会显示一个通知，表明它正在运行。确保这个通知没有被意外关闭或隐藏。这个通知有助于让系统认为Termux是一个活动的应用。

**3. Root 权限专属方法 (效果更强)**

*   **`termux-services` (推荐用于后台服务)**:
    如果你的程序是类似服务的应用（例如SSH服务器、Web服务器），可以使用 `termux-services`。
    ```bash
    pkg install termux-services
    # 假设你要运行 sshd 服务
    # 1. 安装 openssh: pkg install openssh
    # 2. 创建服务目录 (如果服务脚本支持 termux-services)
    #    通常服务包安装后会自动创建，例如 sshd:
    #    mkdir -p ~/.termux/service/sshd
    #    ln -s $PREFIX/share/termux-services/svlogger $PREFIX/etc/service/sshd/log/run # 可选，用于日志
    #    echo '#!/data/data/com.termux/files/usr/bin/sh' > ~/.termux/service/sshd/run
    #    echo 'exec sshd -D' >> ~/.termux/service/sshd/run
    #    chmod +x ~/.termux/service/sshd/run
    # 3. 启用并启动服务
    sv-enable sshd
    sv up sshd
    # 查看状态
    sv status sshd
    ```
    `termux-services` 会尝试保持服务运行。

*   **调整进程的 `oom_score_adj` (高级，需谨慎)**:
    `oom_score_adj` 是Linux内核用来决定在内存不足时杀死哪个进程的值。范围从 -1000 (从不杀死) 到 1000 (优先杀死)。普通应用通常是0或更高。
    你可以用Root权限将Termux主进程及其子进程的 `oom_score_adj` 设置为一个非常低的值。

    1.  找到Termux的进程ID (PID) 或你希望保护的特定子进程的PID。
        *   打开一个新的Termux会话。
        *   运行 `ps -ef | grep termux` (查找Termux相关的进程) 或者 `pgrep -f your_app_name` (查找你的特定应用进程)。
    2.  修改 `oom_score_adj`:
        ```bash
        su
        # 假设Termux主进程或你的目标进程PID是 12345
        echo -1000 > /proc/12345/oom_score_adj
        ```
        **警告**:
        *   `-1000` 意味着内核几乎不会杀死这个进程。如果这个进程失控并消耗大量内存，可能导致整个系统非常卡顿甚至崩溃，因为系统无法通过杀死它来回收内存。
        *   你可能需要为Termux的主进程以及它启动的关键子进程都设置这个值。
        *   更稳妥的做法是设置一个相对较低但不是-1000的值，例如 `-500` 或 `-800`。
        *   这个设置在进程重启后会失效，所以你可能需要写一个脚本，在你的应用启动时自动设置它。

*   **禁用特定厂商的省电/后台管理 (因设备而异)**:
    很多国产手机如小米(MIUI神隐模式)、华为(后台冻结)、OPPO/VIVO等都有自己独特的后台管理机制。你需要：
    1.  在手机的 设置 -> 电池 -> 应用智能省电 (或类似名称) 中，找到Termux，设置为“允许后台运行”、“无限制”或关闭所有优化。
    2.  有些厂商提供了更深层的“锁住后台”功能，通常在最近任务列表中长按应用卡片可以找到。
    3.  对于Root用户，有时可以通过修改系统配置文件或使用特定Magisk模块来彻底禁用这些限制，但这风险较高且具体方法因ROM而异。例如，某些模块如 "Universal GMS Doze" (虽然主要针对GMS，但原理可借鉴) 或针对特定品牌的后台管理禁用模块。

*   **App Freezer / Systemizer (Magisk模块)**:
    *   **App Freezer**: 可以冻结其他不必要的应用，为Termux腾出更多资源。
    *   **App Systemizer (Terminal App Systemizer)**: 将Termux转换为一个系统应用。系统应用通常有更高的运行权限和更低的被杀优先级。**谨慎操作，备份数据。** 如果Termux行为异常，可能需要卸载更新或恢复。

**综合策略**

一个比较稳妥且有效的组合拳通常是：

1.  **Termux**: 使用 `tmux` 或 `screen` 来管理你的进程会话。
2.  **安卓设置**: 将Termux加入电池优化白名单。
3.  **`termux-services`**: 如果是后台服务类应用，优先使用 `termux-services`。
4.  **Root**:
    *   如果上述方法还不够，考虑用 `su` 启动一个脚本，该脚本启动你的应用，并立即将其 `oom_score_adj` 设置为一个较低的值 (例如 `-800`)。
    *   例如，创建一个脚本 `~/start_my_app.sh`:
        ```bash
        #!/data/data/com.termux/files/usr/bin/bash
        # 启动你的应用，确保它在后台运行
        your_command_to_start_app &
        APP_PID=$! # 获取刚启动的后台应用的PID

        # 等待一小会儿确保进程已完全启动 (可选)
        sleep 2

        # 以root权限修改oom_score_adj
        echo "Setting oom_score_adj for PID $APP_PID"
        sudo echo -800 > /proc/$APP_PID/oom_score_adj # 如果你已经su了，就不需要sudo
        # 如果是在 su shell 里执行这个脚本，直接用:
        # echo -800 > /proc/$APP_PID/oom_score_adj

        echo "App started with PID $APP_PID and oom_score_adj set."
        wait $APP_PID # 可选，让脚本等待应用结束
        ```
        然后运行: `su -c "sh /data/data/com.termux/files/home/start_my_app.sh"`
        或者先 `su`，然后在root shell里运行 `sh ~/start_my_app.sh`。

**重要提示**:

*   **内存**: 安卓设备内存有限。如果你的应用本身消耗内存过大，即使有Root权限，系统在极端情况下也可能杀死它以保证基本运行。
*   **稳定性**: 修改 `oom_score_adj` 可能会影响系统稳定性，请谨慎使用。
*   **不同设备差异**: 不同安卓版本和厂商定制ROM的行为差异很大，可能需要尝试多种方法。

先从最简单的方法开始尝试，逐步增加强度，看看哪种最适合你的情况。
