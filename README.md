# 水准测量
本软件由徐晨昊2054153使用C#及Unity引擎独立开发，仅供测量实习使用。

## 使用者文档
软件位于 */软件/水准测量.exe* ，打开软件后，请按下列步骤进行操作：
### 1.输入正确的测站数量
### 2.导入文件，文件内容格式请参考 */示例文件/输入文件.xlsx* 
### 3.检核窗口将自动进行计算，如有超限，将会在右上角铃铛处弹出警告
### 4.在确保输入无误后，点击导出结果，文件内容格式将如 */示例文件/输出文件.xlsx* 所示
### 5.有超限的进行输出后会在特定格子显示
### 6.有任何错误发生时请重新按照1-5的步骤进行，数据将自动清空

## 开发者文档
代码及UI素材源文件位于 */源文件* 中，您可直接打开 */源文件/测量实习.Sln* 参考代码。
下面将讲解各脚本的使用方法：
### 文件IO
为提高软件复用性，必须实现文件IO的操作。文件操作位于`XCH.SurveyPractice.FileOperation`的命名空间中，代码可以复用至任何需要文件读写的软件里。
#### */源文件/Assets/DataCollection/OpenFileName.cs* 

