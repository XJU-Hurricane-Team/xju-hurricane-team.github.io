# 新疆大学飓风机器人文档站

基于[MkDocs](https://www.mkdocs.org/)构建。

感谢[Power's Wiki](https://wiki-power.com/)提供的网站模板。

文档在`docs/zh/`下。

# 参与者如何编辑

注意：每次编辑前请先执行`git pull`保证当前仓库为最新状态以防止产生冲突。

1. 使用`git clone https://github.com/XJU-Hurricane-Team/xju-hurricane-team.github.io.git`将仓库克隆到本地。

2. 确保安装了`Python`，输入`pip install -r requirements.txt`安装Python依赖（如不需要本地使用`mkdosc`，此步骤忽略）。

3. 在`.\docs\zh`路径下的打开/创建文档进行编辑，如有创建文件请在`mkdosc.yaml`文件的`nav`选项下添加文件路径。

4. 编辑完成后提交并**push到main分支**。

5. ~~在终端输入`mkdocs gh-deploy`更新网站。~~`GitHub`会自行将内容同步到网站。

如果对Git不熟悉，可以直接在Github网页编辑markdown文档，我们帮你更新。

# 非参与者如何编辑

1. Fork到您自己的仓库下

2. 使用`git clone 您的仓库地址`将仓库克隆到本地

3. 输入`pip install -r requirements.txt`安装Python依赖（如果已经安装过，此步骤忽略）

4. 编辑完成后提交并push到main分支（一般默认fork的是main分支）

5. 拉取PR，我们帮您合并更新

如果对Git不熟悉，可以直接在Github网页编辑markdown文档，我们帮你更新。
