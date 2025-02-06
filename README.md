# article.wangdm.cn
网摘

```shell
hugo new site article.wangdm.cn
cd article.wangdm.cn
git init
git submodule add https://github.com/wdmsite/LoveIt.git themes/LoveIt
echo "theme = 'LoveIt'" >> hugo.toml
hugo server
```