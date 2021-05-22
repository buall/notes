下载Nginx配置文件的语法文件`nginx.vim`

```bash
wget http://www.vim.org/scripts/download_script.php?src_id=14376 -O /usr/share/vim/vim74/syntax/nginx.vim
```

更改配置文件 `vim /usr/share/vim/vim74/filetype.vim` 。`/etc/nginx/*,/usr/local/nginx/conf/*`这里可以根据Nginx实际安装目录进行修改

```bash
echo 'au BufRead,BufNewFile /etc/nginx/*,/usr/local/nginx/conf/* if &ft == '' | setfiletype nginx | endif' >> /usr/share/vim/vim74/filetype.vim
```

