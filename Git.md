# Git

git config --global user.name "wctang"
git config --global user.email "wctang@gmail.com"
git config --global http.sslverify false


$(prefix)/etc/gitconfig                              # System-wide configuration file.
$XDG_CONFIG_HOME/git/config          # It is a good idea not to create this file
~/.gitconfig                                               # User-specific configuration file. Also called "global" configuration file.
$GIT_DIR/config                                      # (local) Repository specific configuration file.
$GIT_DIR/config.worktree                       # optional, only when extensions.worktree is present in $GIT_DIR/config.



## crlf
https://adaptivepatchwork.com/2012/03/01/mind-the-end-of-your-line/
http://schacon.github.io/git/gitattributes.html

git 對 text 文件會試圖處理 crlf，binary 文件不處理.
(預設判斷 binary 的方法是看前 8000 bytes 內有沒有 NUL 字元. 非 binary 就是 text. link)


Old system: config + .gitattributes
這些設定只處理 “text" 文件

core.autocrlf true   # 提交时转换为LF，检出时转换为CRLF
core.autocrlf input  # 提交时转换为LF，检出时不转换
core.autocrlf false  # 提交检出均不转换

core.safecrlf true   # 拒绝提交包含混合换行符的文件
core.safecrlf false  # 允许提交包含混合换行符的文件
core.safecrlf warn   # 提交包含混合换行符的文件时给出警告

以下不應使用，應用新格式
.gitattributes       # 設定各式檔案是否轉換
[
*.txt crlf
*.jpg -crlf
*.abc crlf=input
]


New System: .gitattributes text  (Git 1.7.2 and above.)
可用 .gitattributes 定義，優先於個人設定
如果 .gitattributes 內無定義，則 fall back 至個人 core.autocrlf 的設定 (舊系統)
This is done with a text attribute in your repository’s .gitattributes file. 
設定值:
text: 提交時 CRLF->LF, 檢出時 LF->CRLF
-text: 提交檢出均不轉換
text=auto: "如果是 text 時"提交時 CRLF->LF, 檢出時 LF->CRLF (依賴 git 的 binary 判斷機制)
eol=


eol
This attribute sets a specific line-ending style to be used in the working directory. It enables end-of-line normalization without any content checks, effectively setting the text attribute.
Set to string value "crlf"
This setting forces git to normalize line endings for this file on checkin and convert them to CRLF when the file is checked out.
Set to string value "lf"
This setting forces git to normalize line endings to LF on checkin and prevents conversion to CRLF when the file is checked out.
Backwards compatibility with crlf attribute
For backwards compatibility, the crlf attribute is interpreted as follows:
crlf            text
-crlf           -text
crlf=input      eol=lf
End-of-line conversion
While git normally leaves file contents alone, it can be configured to normalize line endings to LF in the repository and, optionally, to convert them to CRLF when files are checked out.
Here is an example that will make git normalize .txt, .vcproj and .sh files, ensure that .vcproj files have CRLF and .sh files have LF in the working directory, and prevent .jpg files from being normalized regardless of their content.
*.txt           text
*.vcproj        eol=crlf
*.sh            eol=lf

# Images should be treated as binary
# (binary is a macro for -text -diff -merge)
*.png     binary
*.jpg      binary
*.jepg    binary
*.sdf      binary


重整 crlf 的設定
commit any change before do this.
change autocrlf, gitattributes...
git add --renormalize .
git commit -m "Introduce end-of-line normalization"
git read-tree --empty
git reset --hard master











http://stackoverflow.com/questions/9392365/how-would-git-handle-a-sha-1-collision-on-a-blob

# 觀念

working-tree (working directory)
: 工作區

repository
: 儲存庫

stage (index)
: 下次將會 commit 的 change set
- stage 中檔案的內容不在 index 檔，而是會放在 .git/objects 裡 (所以 add 後就已進入 objects 中，而非 commit 後)
    記錄 working-tree 中檔案的 timestamp/長度, 用以加速如 status, diff 等命令的執行
    實際上不是 tree, 而是個 flattened manifest.
    http://blog.ossxp.com/2010/11/2166/

~~~~~~
      <working-tree> 
     ---------------------------------- 
         <index>       <HEAD>       .git
                         |
         <commit> <--- <branch>
            |
         <commit> 
         /      \
    <commit>  <commit>
       |     /
    <commit>
~~~~~~

- 4 種物件: blob, tree, commit, annotated tag, 每個都有 sha1, type, size, content
- 指標 (ref): branch, lightweight tag, HEAD

HEAD
:   通常代表最近的一次提交, next parent. 現在 working-tree 對應的 head, commit 會加入的地方. 一般來說 HEAD 會是 ref to branch, 不然就是 detached HEAD

- 在 packs 中會用 delta 的方式存放
- git 在 local clone 時會使用 hard link.


# 慣用法
- 用 "`--`" 分隔 commit 和 path (以免名稱相同)
- 指令中 commit 未指定則為 HEAD

~~~~~~
  A       HEAD
  |
  B       HEAD~1 = HEAD~ = HEAD^1 = HEAD^
  | \
  |  C    HEAD~1^2 
  D  |    HEAD~~ = HEAD~2 = HEAD~1^1 = HEAD^^
  | /
  E       HEAD~~~ = HEAD~3 = HEAD~2^1
~~~~~~

- 版本範圍 "`A..B`": (B 向上走到底) 減去 (A 向上走到底)
    - 如果 A 是 B 的父代，`A..B` = A 到 B 間的版本，"不包括 A"
    - 如果 A 是 B 的兄弟，`A..B` = B 向上走到和 A 的共同父代，"不包括這個父代"

# 使用
## 常用指令 & 反操作

~~~~~~
             "修改"               "add <files>"               "commit"
[未修改] -------------->  [已修改] --------------> [staged] --------------> [commited]
       <--------------          <--------------          <--------------
    "checkout -- <files>"       "reset -- <files>"       "reset --soft HEAD^"
        <--------------------------------------
                 "reset --hard HEAD"
        <---------------------------------------------------------------
                              "reset --hard HEAD^"
~~~~~~

git init --bare
:   server 的 repository 要用 `--bare` 建立，以避免 push 進 woriking-tree 造成的 warning.    

git add -p <files>
:   交互式選擇要加入的 hunks

git status
:   查看 working-tree / stage 的內容 

## checkout

修改 HEAD 所指向的 分支

`git checkout <branch>`
:   切換 branch. (將 HEAD 指向 <branch>, 更新 stage & working-tree)
    (當 working-tree 有修改 & 切換後有衝突就失敗, 如果 branch = HEAD 就無作用) (**移動 HEAD**)
`git checkout -b newbranch [<start_point>]`
:   新增一個 newbranch = commit 並切換至 newbranch. 如果沒有 -b 則變成 detached HEAD.
`git checkout [<tree-ish>] -- <pathspec>...`
:   由 "stage" 回復 working-tree 的檔案。如果有指定 tree-ish, 會先由 tree-ish 更新 stage, 再更新 working-tree. (**不移動 HEAD**)


## reset

修改 目前分支 所指向的 branch

`git reset [<commit>] -- <paths>...`
:   "un-stage", git add 的反操作, 將 stage 中的 paths 還原為 commit HEAD 中的版本, 不影響 working-tree & HEAD.

`git reset (--soft | --mixed | --hard) [-q] [<commit>]`
:   移動 HEAD 所指向的 branch 至指定的 commit (detached HEAD 則是單純移動 HEAD), 並且：

    - `--soft`    # 不重置 stage / working-tree
    - (`--mixed`) # 重置 stage, 但不重置 working-tree. (un-stage)
    - `--hard`    # **!!危險!!** 重置 stage 和 working-tree

常用法:

- `git reset --soft HEAD^` # 取消前一次 commit，保留 stage/working-tree, 可再重新 commit 一次.
- `git reset --hard` # **!!危險!!** 重置 stage & working-tree 為 HEAD 內容 = 還原成修改前，放棄修改
- `git reset --hard HEAD^` # **!!危險!!** 取消前一次 commit，且不保留 stage/working-tree (可用 git reset --hard HEAD@{1} 再回復)


比較 git reset & git checkout

- `git checkout $id` # 讓 HEAD 改指向 $id, 變成 detached HEAD
- `git reset $id` # 把 (HEAD 所指的) master (或某個 branch) 移動至 $id, 而 HEAD 仍指向 master, 如果 HEAD 已是 detached, 那就和 checkout 相同 (但有修改不會警告)

<http://progit.org/2011/07/11/reset.html>

## revert

`git revert <commit>` 
:   將要撤銷的該次 commit 的修改反轉，然後和 working-tree 做 merge, 會形成一次 rollback commit.


## diff

`git diff`
:   比較 working-tree & stage (已修改，未 add 的部分)
`git diff --staged`
:   比較 stage & HEAD (已 add，未 commit 的部分)
`git diff <commmit1> <commit2> -- <path>`
:   比較 path 在 commit1 和 commit2 間的差別


## stash

實際上也是存在 objects 中，用 refs/stash 指向

`git stash`
:    
`git stash pop`
:    


## tag
`git tag [tagname]`
:   lightweight tag (refs/tags/tagname 指向 commit object), commit 的別名

`git tag -a [tagname]`
:   annotated tag (refs/tags/tagname 指向 tag object), 可追蹤建立者及建立時間, 加上說明

`tag push origin --tags [tagname]`
:   tag 在 fetch 會自動抓取，但 push 時並不會自動上推，要手動 push 才會出去

`git push origin :[tagname]`
:   刪除遠端的 tag

- 要移動已存在的 tag -> 重新加入同名的 tag

`git pull origin refs/tags/xxxxx`
:   更新 tag (已有同名 tag, 就不會再從上游同步 (不會自動跟隨)，要手動更新才行)


## branch

commit object 的 指標 (ref).

`git branch xxx` # create branch
:    
`git branch -r` # show remote branch
:    
`git branch --track <branch name> <remote branch>` # 建立 branch 並設定 tracking branch
:    


## merge

自動 merge 失敗，git 會進入 MERGING mode 等待解決 conflict, conflict file 會被標上 mark (git status 會顯示 unmerged)
解決後 git add ，然後 git commit 就可解決 conflict 並離開 MERGING 模式。MERGING mode 不可切換至其他 branch, 除非放棄 merge.
中途放棄 merge: git reset --hard HEAD
* git merge [branch] // 如果沒有衝突，會直接 commit
* git merge --no-ff [branch] // 不要 fast forward
* git merge --squash [branch] // 把被 merge 進來的 branch 縮成一次 commit

git blame <path>
    逐行列出每一行是由誰最後修改


## remote

remote repository / branch
* git clone <repo uri> // 會 track "所有的" remote branch, 然後 checkout HEAD (不一定是 master) branch.
* git remote add xxx git://xxx.xx.xx.x/xxx.git // 為 remote 取名
* git branch --track exp origin/exp // 建立一個 tracking branch to origin/exp
* git branch --set-upstream exp origin/exp // 設定已存在的 branch 為 tracking branch to origin/exp
* git fetch
* git pull          = git fetch + git merge
* git pull --rebase = git fetch + git rebase
* git push
* tracking branch: is a local branch that is connected to a "remote branch"
* 對 tracking branch 做 push/pull 會自動對其對應的 remote branch 做 push/pull.
* clone 時會設定 master 是 tracking branch to origin/master
* 不能直接修改或 local commit 
* stored in .git/refs/remotes/
* remote 只能進行 fast-forward, push 時不會新增 commit (不會merge)，所以當 push 時, remote 和 local 端不相容，必需先 fetch, merge, 再 push，不會自動進行
* 因為單純用 git push, 預設的行為是把自己所有和遠端有 tracking 關係的 branch 都 push，不一定是想要的行為，所以會有 warning。所以最好能設定 git config (--global) push.default=current 來只 push current branch。
* 位於 local new branch 做 push 就會對 remote 新增 branch
* 使用 git push origin :<branch name> 來刪除遠端 branch
* 如果已 fetch 過的 remote branch 在遠端被刪除了的話，在local端不會受影響
	* 如果要刪除的話，用 git branch -rd origin/<branch name> 刪除
* git push -u origin <local_branch> : 將 local_branch push 至遠端，並自動設定 tracking.


## rebase

`git rebase [-i] [--onto <newbase>|<since>] <since> [<till>|HEAD]`
:   將 <since>..<till> 剪下，接到 <newbase> 上. (注意不包括 <since>, -i = 互動式 rebase, 把 <since>..<till> 列出，選擇性 apply)

Linus: "Eventually you’ll discover the Easter egg in Git: all meaningful operations can be expressed in terms of the rebase command."

~~~~~~
    eg: A -- B -- C -- D -- E -- F(HEAD)
              \-- G -- I(master)

        git rebase master
        A -- B -- G -- I -- C' -- D' -- E' -- F(master,HEAD)

        git rebase --onto B C
        A -- B -- D' -- E' -- F(HEAD)
              \-- G  -- I(master)

        git rebase --onto G C E
        A -- B -- G -- I(master)
                   \-- D' -- E'(HEAD)
~~~~~~

* git rebase -i HEAD~3 // 將範圍內的 commit 進入選取階段



## 修改歷史版本 (常需和 reset 合併使用) 

`git commit --amend`
:   修改最後的一次 commit. 將前一次 commit 和現在的 stage 合併，並重新 commit

`git cherry-pick <commit>...`
:   將 <commit> 的變化 apply 並 commit.





# 合併衝突
* 當 merge 發生 conflict 時，會將 3 個版本以不同的編號放在 stage 中 (沒有 conflict 的檔案編號 = 0, git ls-files -s)
    1. conflict 雙方共同的祖先 (git show :1:filename)
    2. 執行 merge 本身的版本 (git show :2:filename)
    3. 試圖被 merge 進來的版本 (git show :3:filename)
    而 working-tree 中放著自動 merge 的結果.
* 修改解決 conflict 後，執行 git add 把 merge 結果放入 stage (編號 0, 此時會把其他編號的版本刪除), 再 commit 就解決 conflict 了


# 進階指令

`reflog`
:   HEAD 移動的記錄, 記錄 branch tips 變化的情況


# 底層指令
rev-parse
    列出指定 object 的 SHA1. 和有的沒的功能
rev-list
    版本範圍查找
ls-tree
write-tree
ls-files
cat-file


# submodules
// 母 repo 只是記錄 sub repo 的 commit id, 在母repo 用 pull 無法更新 sub repo，仍然必需進入各別目錄執行 pull
// sub repo 修改後 (commit id 變化)，母 repo 用 status 可以看到 sub repo 有變，但必需自已執行一次 commit 才能跟隨 sub repo.
// sumodule update 後，sub repo 的 HEAD 是處於 detached HEAD 的情況(因為是指定特定 commit id)，而不是跟隨某個 branch.
// 其實很難用...
the submodule support just stores the submodule repository location and commit ID.













# 比較

## v.s. Mercurial

- http://felipec.wordpress.com/2011/01/16/mercurial-vs-git-its-all-in-the-branches/
- http://stackoverflow.com/questions/1598759/git-and-mercurial-compare-and-contrast
- http://blog.ossxp.com/2011/03/2370/
- 主要差異在 branch.
    - git branch: refs, pointer to commit.
	- mercurial branch: linear sequence of changesets. "default" is the default branch name. branch name is an attribute associated with a changeset.

## v.s. subversion

- svn 在 merge 時記錄的東西不足，以至於要指定 merge 的 reversion 區間，否則會要重覆解 conflict
- 所有的東西在 merge 時匯入，無法分辨單純為了解 conflict 的改變
 

# 參考資料

-Linus Torvalds on git <http://www.youtube.com/watch?v=4XpnKHJAok8>
    - Every developer's working-tree is a branch.
    - Every single person has it's own branch. It's not horrible, it just is.
    - Every body have commit access, on it's own branch.
    - Subversion make branch very cheap, but who care? branch is complete useless unless you merge them. branching is not the issue, merging is.
- A Visual Git Reference http://marklodato.github.com/visual-git-guide/index-en.html
- Pro Git: http://progit.org/book/
- The Git Community Book: http://book.git-scm.com/index.html
- Git Magic: http://www-cs-students.stanford.edu/~blynn/gitmagic/
- git ready: http://gitready.com/

- http://gitstory.wordpress.com/category/git/

- Getting Git http://gitcasts.com/posts/railsconf-git-talk
- Git the basics http://excess.org/article/2008/07/ogre-git-tutorial/
- Randal Schwartz on git  http://www.youtube.com/watch?v=8dhZ9BXQgc4

- http://linux.chinaunix.net/bbs/thread-920610-1-1.html
- http://www.kernel.org/pub/software/scm/git/docs/everyday.html

- Branching and merging with git: http://lwn.net/Articles/210045/

- http://git.or.cz/gitwiki/GitSvnComparsion

- 看日记学git http://roclinux.cn/?cat=72

- http://www.vogella.com/articles/Git/article.html

- http://www.dribin.org/dave/blog/archives/2007/12/28/dvcs/
- http://docs.google.com/Present?docid=ddwtzk7_10877cgr7hwhq
- http://sayspy.blogspot.com/2006/11/bazaar-vs-mercurial-unscientific.html

- Glossary http://git.or.cz/gitwiki/GitGlossary



## Git (msysGit) on windows
1. 安裝 Git-xxxxxxxxxxx.exe
    1. Use Git Bash only
    2. Checkout windows-style, commit Unix-style line encodings
2. [~/.bashrc
export LESSCHARSET=utf-8

alias ls='ls --show-control-chars --color=auto'
]
3. [/etc/inputrc
set meta-flag on
set input-meta on
set output-meta on
set convert-meta off
]
4.[~/.gitconfig
[gui]
    encoding = utf-8
[i18n]
    commitencoding = utf-8
    logoutputencoding = cp950
[core]
    symlinks = false
    autocrlf = true
[color]
    ui = auto
]


# gitweb

http://weininger.net/configuration-of-nginx-for-gitweb-and-git-http-backend.html

apt-get install git
apt-get install nginx
apt-get install fcgiwrap

sudo adduser git

// put all git repos to  /home/git/repos/git
mkdir -p /home/git/repos/git
chmod -R g+ws /home/git/repos/git


[/home/git/gitweb.nginx
server {
    listen 80;
#    server_name git.server.name;

    client_max_body_size 500M;


    # static repo files for cloning over https
    location ~ ^/git/.*\.git/objects/([0-9a-f]+/[0-9a-f]+|pack/pack-[0-9a-f]+.(pack|idx))$ {
        root /home/git/repos/;
    }

    # requests that need to go to git-http-backend
    location ~ ^/git/.*\.git/(HEAD|info/refs|objects/info/.*|git-(upload|receive)-pack)$ {
        root /home/git/repos;

        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        fastcgi_param SCRIPT_FILENAME /usr/lib/git-core/git-http-backend;
        fastcgi_param PATH_INFO $uri;
        fastcgi_param GIT_PROJECT_ROOT /home/git/repos;
        fastcgi_param GIT_HTTP_EXPORT_ALL "";
        # fastcgi_param REMOTE_USER $remote_user;
        include fastcgi_params;
    }

    # gitweb website
    location /gitweb/static {
        root /usr/share;
    }

    location ~ ^/git/(.*)$ {
        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        fastcgi_param SCRIPT_FILENAME /usr/share/gitweb/gitweb.cgi;
        fastcgi_param PATH_INFO $1;
        fastcgi_param GITWEB_CONFIG /etc/gitweb.conf;
        include fastcgi_params;
   }
}
]
sudo ln -s /home/git/gitweb.nginx /etc/nginx/sites-enabled/

[/home/git/gitweb.conf
# path to git projects
$projectroot = "/home/git/repos/git/";

# directory to use for temp files
$git_temp = "/tmp";

# target of the home link on top of all pages
$my_uri = "/git/";

$stylesheet = "/gitweb/static/gitweb.css";
$javascript = "/gitweb/static/gitweb.js";
$logo = "/gitweb/static/git-logo.png";

# enable nicer uris
$feature{pathinfo}{default} = [1];
]
sudo ln -s /home/git/gitweb.conf /etc/




create repo:
su as git
mkdir xxx.git
git init --bare xxx.git
echo "[http]" >> xxxx.git/config
echo "        receivepack = true" >> xxxx.git/config
sudo chown -R www-data xxxx.git



------------------ enable ssl and auth
nginx

[~git/gitweb.nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
#    server_name git.server.name;

    ssl on;
    ssl_certificate /home/git/server.crt;
    ssl_certificate_key /home/git/server.key;

    root /usr/share/gitweb;
...
]

------- enable auth
sudo apt-get install apache2-utils

sudo htpasswd -c ~git/htpasswd xxxx
(enter password)

[~git/gitweb.nginx
...
    ssl_certificate_key /home/git/server.key;
    ssl_session_timeout  5m;

…

    location ~ ^/git/.*\.git/objects/([0-9a-f]+/[0-9a-f]+|pack/pack-[0-9a-f]+.(pack|idx))$ {
        root /home/git/repos/;
       auth_basic "Restricted access";
       auth_basic_user_file /home/git/htpasswd;

...
]



	
-------

# 安裝 gitweb + git service on ubuntu

* 安裝 git-core, gitweb, apache2
* 設定 GIT_PROJECT_ROOT
[/etc/apache2/conf.d/gitweb

SetEnv GIT_PROJECT_ROOT /home/.repo
SetEnv GIT_HTTP_EXPORT_ALL

ScriptAliasMatch \
	"(?x)^/git/(.*/(HEAD | info/refs | objects/(info/[^/]+ | [0-9a-f]{2}/[0-9a-f]{38} | pack/pack-[0-9a-f]{40}\.(pack|idx)) | git-(upload|receive)-pack))$" \
	/usr/lib/git-core/git-http-backend/$1

Alias /git /usr/share/gitweb
<Directory /usr/share/gitweb>
  Options FollowSymLinks +ExecCGI
  AddHandler cgi-script .cgi
</Directory>
]
* 設定 $projectroot
[/etc/gitweb.conf
$projectroot = "/home/.repo";
...
]
* sudo service apache2 restart

如此可以連至 http://xxx.xxx.xxx.xxx/git/ 至管理頁面
而 push / pull 是 http://xxx.xxx.xxx.xxx/git/xxx.git

* 要注意在 /home/.repo/xxx.git 要 chown 為 www-data, 用 git init --bare
* 要在 repo 的 conf 中加上
[/home/.repo/xxx.git/conf
[http]
	receivepack = true
]
才能 push, 不然只能 fetch, pull 而已









------
cgit (not ok…)

apt-get install curl libssl-dev

wget http://git.zx2c4.com/cgit/snapshot/cgit-0.10.2.tar.xz
tar xf cgit-0.10.2.tar.xz

make get-git
make CGIT_SCRIPT_PATH=/home/wctang/www prefix=/home/wctang/cgit
make CGIT_SCRIPT_PATH=/home/wctang/www prefix=/home/wctang/cgit install


server {
    listen       80;
#    server_name  git.server.name;

    root /home/wctang/www;

    # send anything else to gitweb if it's not a real file
    try_files $uri @cgit;
    location @cgit {
        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        fastcgi_param SCRIPT_FILENAME   /home/wctang/www/cgit.cgi;
        fastcgi_param PATH_INFO         $uri;
        fastcgi_param QUERY_STRING    $args;
        include fastcgi_params;
   }
}




