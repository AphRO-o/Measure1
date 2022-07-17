# 导线测量
  本软件由徐晨昊2054153使用C#及Unity引擎独立开发，仅供测量实习使用。
  本软件可以进行闭合导线的坐标计算。
  
-----

## 使用者文档
  软件位于 ***/软件DEMO.exe*** ，打开软件后，请按下列步骤进行操作：
  
1.在右侧输入栏输入正确的测站数量、起算方位角与起算坐标；

2.输入数据。您可以在左侧的输入框内手动输入，也可以点击导入文件按钮，文件内容格式请参考 ***/输入输出示例/输入文件.xlsx*** ；

3.根据需要进行数据修改，或者点击添加扰动按钮使所有数据产生微小浮动。每一次更改后，程序都将自动进行计算，并将结果显示在右下的信息栏内。如有超限，将会在右上角铃铛处弹出警告；

4.在确保更改结束后，点击导出结果到指定位置，文件内容格式将如 ***/输入输出示例/输出文件.xlsx*** 所示；

5.输出的Excel文件中，所有对数据的更改都将保留，检核计算与超限提醒将在特定格子显示，用户根据自己的需要截取数据；

6.由于制作仓促，难免有难以发现的错误。遇到错误发生时请点击重置按钮或重启软件，并重新从步骤1开始操作。

-----

## 开发者文档

代码及UI素材源文件位于 ***/源文件*** 中，您可直接打开 ***/源文件/测量实习.Sln*** 参考代码。

### 下面将讲解各脚本的使用方法：（详细参见各脚本注释）

#### 1.数据库
  本软件使用了Unity命名空间下的ScriptableObject类将数据统一存储在Asset资源包中，便于其他脚本统一访问。***/源文件/Assets/DataCollection/DataDetails.cs*** 文件定义了一个测站需要得到的所有数据，并以测站为一个数据块的单位，将这些数据块存储在 ***/源文件/Assets/DataCollection/Data_SO.cs*** 声明的List数据库中。

#### 2.文件开闭
  为提高软件复用性，必须实现文件IO的操作。文件操作位于`XCH.SurveyPractice.FileOperation`的命名空间中，代码可以复用至任何需要文件读写的软件里。***/源文件/Assets/DataCollection/OpenFileName.cs*** 定义了读写文件的数据结构；***/源文件/Assets/Scripts/LocalDialog.cs*** 中实现了以系统调用方式呼出“打开文件”与“保存文件”的Windows窗体； ***/源文件/Assets/Scripts/SelectFile.cs*** 则使用这些方法与UI按钮绑定，记录用户读写的位置，便于后面对Excel文件进行打开与保存的操作。
  
#### 3.读写Excel
  本软件使用了EPPlus类库读写Excel表格。在获得用户选择的路径后，***/源文件/Assets/Scripts/ContactWithExcel.cs*** 将根据该路径打开对应的Excel文件，并按既定的格式读取内容至数据库中，与此同时，完成一个测站内所有数据的计算，如后视距、视距差、前视距、前后累积差、黑色/红色面后尺减前尺、平均高差等。在用户导入完毕即将导出时，该脚本将数据库中的数据重新写回Excel，并可根据用户需要另存为至任意位置。
  
#### 4.数据处理与验核
  所有测站内部的数据（除改正后平均高差）都在读入的时候就计算完毕，然后由 ***/源文件/Assets/Scrips/DataManager.cs*** 接管这些数据，并做一些必要的加和处理，如计算高差闭合差、视距和、总距离等，并依此得到高差闭合差的距离分配系数，计算得到各测站的改正后高差。检核也在此处完成，先针对各测站的视线长、前后视距离差、红黑面读数差和红黑面高差之差做判断，之后再总体上判断环路闭合差与前后视距离累计差是否符合条件，代码如下所示：
 ```
   private void CheckLimit()
    {
        foreach(var data in data_SO.dataList)
        {
            if (data.Parallax > 5 || data.RearLevelCheck > 3 || data.FrontLevelCheck > 3 || data.RearFrontDifferenceLevelCheck > 5 || data.TotalSightDistance > 80)
            {
                EventHandler.CallOverLimitEvent();
                isOverLimit = true;
            }
        }

        if (!isOverLimit)
        {
            float differenceLimit = 20 * MathF.Sqrt(distance * 0.001f);
            if (heightDifferenceSum * 1000 > differenceLimit || data_SO.dataList[spotCount - 1].ParallaxSum > 10)
            {
                EventHandler.CallOverLimitEvent();
                isOverLimit = true;
            }
        }
    }
  ```
  如有超限的数据，***/源文件/Assets/Scrips/DataManager.cs*** 将通过 ***/源文件/Assets/Scripts/EventHandler.cs*** 通知 ***/源文件/Assets/Scripts/UIManager.cs***，后者将在屏幕上弹出超限警告，此时输出文件，特定单元格中同样将提示超限。
  
#### 5.工具类
  ***/源文件/Assets/Scripts/EventHandler.cs*** 作为全局静态类，是各事件的事件中心，其目的是充分解耦，增强软件的可扩展性，未来的所有事件都可在此声明。
  ***/源文件/Assets/Scripts/Singleton.cs*** 泛型单例类，一些manager文件全局仅存在一个，使他们成为单例便于其他类进行访问调用。
