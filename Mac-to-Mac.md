# iTerm2 Mac-to-Mac & Remote Handoff Guide

This guide explains how to use iTerm2's native `tmux` integration (`tmux -CC`) to seamlessly sync your terminal layout, scrollback, and running processes across multiple machines. 

By using `tmux -CC` (Control Mode), iTerm2 replaces tmux's traditional text-based UI with **native macOS windows, tabs, and split panes**. This gives you the persistence of tmux with the native scrolling, shortcuts, and UI of iTerm2.

---

## Scenario 1: The Remote Linux/Unix Dev Server

Imagine you are working on a powerful remote Linux machine (e.g., `dev-server`). You want to set up a complex grid of terminal panes, run long compilations, and be able to disconnect and reconnect from any Mac without losing your layout or interrupting your work.

### How it Works
1. **Connect:** From your Mac, SSH into the remote machine: `ssh user@dev-server`
2. **Start Control Mode:** Run `tmux -CC`. 
3. **Native UI:** iTerm2 will open a new, native macOS window. This window is now "tmux-backed."
4. **Build Your Layout:** Inside this new window, use standard iTerm2 shortcuts (`Cmd+D` for vertical split, `Cmd+Shift+D` for horizontal split, `Cmd+T` for a new tab) to create your workspace. Every pane you create is natively rendered on your Mac but is actually a shell running on `dev-server`.
5. **Handoff:** If you close your laptop, the SSH connection drops, but the processes on `dev-server` keep running. Later, from a different Mac, you can SSH into `dev-server` and run `tmux -CC attach`. iTerm2 will instantly recreate your exact native window layout and restore the scrollback for every pane.

---

## Scenario 2: The Mac-to-Mac "Lakehouse" Workflow

This scenario is for the ultimate power user who treats their primary desktop Mac as a personal cloud server. 

**The Setup:** You have a Mac Studio on your desk connected to two 5K Studio Displays, and you are heading to a lakehouse with your 15" MacBook Air. You want to pick up your exact terminal state—including local Mac Studio processes and active SSH sessions to other servers—from the MacBook Air.

### How it Works
1. **Host Setup (Mac Studio):** 
   - Ensure `tmux` is installed (e.g., via Homebrew: `brew install tmux`).
   - Enable **Remote Login** in macOS System Settings (General > Sharing) to allow SSH connections.
   - Open iTerm2 on the Mac Studio, type `tmux -CC`, and build your workspace across your Studio Displays. Some panes might be running local node servers, while others are SSH'd out to external databases or servers.
2. **Remote Access:** You need a way to reach your Mac Studio's SSH port from the lakehouse. A mesh VPN like [Tailscale](https://tailscale.com) is highly recommended for secure, zero-config access (allowing you to simply `ssh user@mac-studio`).
3. **Client Setup (MacBook Air):** 
   - Open a blank iTerm2 window on your 15" MacBook Air.
   - SSH into your Mac Studio: `ssh user@mac-studio`
   - Run: `tmux -CC attach`

### The Magic
iTerm2 on your MacBook Air will instantly spawn native macOS windows, tabs, and split panes that perfectly mirror your Mac Studio's layout. 
- You are remotely controlling the Mac Studio's local shells.
- If a pane on the Studio was SSH'd into another server, the MBA pane picks right up in that exact shell (MBA -> Mac Studio -> Remote Server). Your SSH keys and connections on the Studio are completely undisturbed.

### Detaching Safely
When you are done working on the MacBook Air and want to leave the sessions running for when you return to the Mac Studio:
1. Press `Esc` to bring up the iTerm2 tmux menu.
2. Select **Detach**.
Your native iTerm2 windows on the MacBook Air will cleanly vanish, but the hidden `tmux` server on the Mac Studio keeps everything running perfectly in the background.

---

## Important Gotchas

### 1. Screen Resolution and Window Sizing
In the Mac-to-Mac scenario, your Mac Studio has massive 5K screens, while your MBA is 15 inches. When you attach from the MBA, tmux forces the internal layout to fit the smallest currently attached screen. iTerm2 handles this gracefully, but your panes will be resized to fit the 15" display. When you return to the Mac Studio and attach, you can simply resize the windows back out to fill your Studio Displays.

### 2. Never Double-Nest `tmux -CC`
If you are inside a `tmux -CC` session (for example, on your Mac Studio) and you SSH into a remote Linux server from one of those panes, **do not run `tmux -CC` again on the remote server**. Control mode cannot be nested inside itself. If you need tmux on the remote server, run standard `tmux` without the `-CC` flag, or rely entirely on the Mac Studio's outer tmux layer to manage your panes.

### 3. Shared Multiplayer
If you leave the Mac Studio logged in and attach from the MacBook Air at the same time, both Macs will share the session. If you type on the MacBook Air, you will see the characters appear in real-time on the Mac Studio's screens! This is fantastic for pair programming or remote troubleshooting.