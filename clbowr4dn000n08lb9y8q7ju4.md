# 6 tools I use every day as a full-stack developer

I know there are lots of posts about "necessary" tools that "you must use" as a developer. Without going that far, here is a short list of tools I use daily I can't personally live without.

## 1\. [Maccy](https://maccy.app) - MacOS clipboard manager

(**⚡⚠⚡ For non Mac OS users** ) I am sure you can find a decent alternative for your operating system. [Windows 10 even comes with a built-in clipboard history](https://www.pcworld.com/article/394506/how-to-use-windows-10s-clipboard-history.html). It is worth searching, because a good clipboard manager can change your life!

Maccy "*does one job - keep your copy history at hand. Period.*" Lightweight and open source, it stores whatever you copy. Need an old entry? Press `⌘+y` (or whatever shortcut you want) to show the menu, either search or use arrows to select what you need, press enter and it is back in your clipboard.

![maccy overview](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098078581/i8Qi91DHI.png align="left")

## 2\. [lazygit](https://github.com/jesseduffield/lazygit) - Git UI in your terminal

Instead of trying to learn how to use the git UI in IntelliJ, only to start again when switching to Visual Studio, I prefer running git in the terminal, which is always here.

Lazygit comes with panels and shortcuts, which are all pretty straightforward and customizable. Once you get the gist, it becomes really fast to do whatever in git.

![lazygit overview](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098080738/rKPg6Ymwz.gif align="left")

Press `a` to stage all files, `c` to commit, `C` to commit using the usual git prompt, `space` to stage/unstage the selected file, etc.

If a shortcut is missing for an action you do often, you can add one easily by creating [custom commands](https://github.com/jesseduffield/lazygit/blob/master/docs/Custom_Command_Keybindings.md) (note: custom commands are written in `~/Library/Application\ Support/lazygit/config.yml` on Mac OS).

## 3\. Rainbow Brackets

![Image description](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098082684/PqgEgfoEH.png align="left")

So simple, yet so helpful. Rainbow brackets colors matching parentheses, brackets, etc. It is hard to look at code without this feature once you get accustomed to it.

There are multiple extensions depending on your IDE:

⮕ [IntelliJ plugin](https://plugins.jetbrains.com/plugin/10080-rainbow-brackets)

⮕ [Visual Studio Code plugin](https://marketplace.visualstudio.com/items?itemName=2gua.rainbow-brackets)

Note that IntelliJ is really feature-rich, even supporting XML tags and the like. For Visual Studio, you may have to install a bunch more extensions to get the same result: https://dev.to/jennifer/when-you-want-a-bit-more-rainbow-in-your-vscode-10aj

## 4\. [Vivaldi](https://vivaldi.com) browser

I have a problem: I am a tab freak. I always have at least 50 tabs opened at the same time, in at least 2 browser windows (one for each screen !).

Vivaldi is a chromium-based browser, with a highly customizable UI written in react, that comes with anti-tracking and Ad Blocker out-of-the-box. But the real plus for me are [the tab management features](https://vivaldi.com/features/tab-management/). My favorites:

*   tab position: top, bottom, left, **right**
    
*   tab cycling (with preview 🤩): either by position, or last usage
    
*   bookmark nicknames usable in the top bar (like firefox)
    
*   ability to create keyboard shortcuts for everything
    
*   spotlight: search open tabs, commands, etc.
    

And other nice ones:

*   tab stacks (grouping tabs), with the ability to split screen between them (multi-tasking)
    
*   tab preview on hover
    
*   beautiful and customizable themes
    

Here is what it looks like with my theme and settings (see how easy it is to find the dev.to tab):

![Vivaldi with tabs on the right](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098084569/ZGy7bJgTj.png align="left")

Note that Vivaldi is a bit slower than Chrome, but I find it a good bargain given the time I spare looking for the right tab. You can also improve the speed by disabling some of the features. Let me know in the comments if you are interested in knowing more.

## 5\. [Typora](https://typora.io)

I am not a particularly organized person: when I take notes, I don't want to think of tags, knowledge graphs, or databases. Simple markdown files and a basic search interface are all I ask for.

![Typora overview](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098087283/v7Xgc5tN4.gif align="left")

[Typora](https://typora.io) is a paid markdown editor, yet its fair price ($ 14.99 for a lifetime license usable on three devices) is totally worth it.

Contrary to most other editors, it doesn't split your screen between write and preview. It is simple yet supports tables, code + code highlighting, latex, and more. You can use regular markdown, or shortcuts and menus to format your text.

The themes and *export to PDF* feature are so good I sometimes used it for my reports when I was still a student.

More importantly, everything is stored locally. I thus can use it to organize my notes on my work machine (without risking leaking sensitive information).

I personally use a *MD* folder where I store all my notes and open the folder itself on typora. I now have a sidebar to browse my files, and a search box supporting fuzzy search. This folder is then synced with my company cloud or Dropbox/Google Drive/... on my personal computer.

It is easy, lightweight, and works well. Even better, my notes are just text files, so I can open them with other softwares if I get tired of typora. No vendor locking !

## 6\. [KeePassXC](https://keepassxc.org)

![KeePassXC](https://cdn.hashnode.com/res/hashnode/image/upload/v1671098089541/F3y680ukT.png align="left")

KeePassXC is an open-source cross-platform password manager that works completely offline.

I mostly use it for work, where I have soo many passwords and 2FAs. What I like about KeePassXC is:

*   databases are stored on files locally (but can be synced using my OneDrive for work),
    
*   supports TOTP,
    
*   can notify you of password expiration (the 3-month policy in many organizations),
    
*   browser integrations, password generation, ...