# 如何通过命令行删除远程tag

项目开发过程中，随着时间的迁移，可能导致仓库中无效的tag越来越多，需要进行清理。可以通过命令行来远程删除。

Or, more expressively, use the --delete option (or -d if your git version is older than 1.8.0):  
  
\- git push --delete origin tagname  
  
If you also need to delete the local tag, use:  
  
\- git tag --delete tagname