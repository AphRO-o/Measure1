# 水准测量
  本软件由 **徐晨昊2054153** 使用 **C#及Unity引擎** 独立开发，仅供测量实习使用。

  本软件用于水准测量高程。
  
-----

## 使用者文档
  软件位于 ***/水准测量DEMO.exe*** ，打开软件后，请按下列步骤进行操作：
  
1.输入正确的测站数量；

2.导入文件，文件内容格式请参考 ***/示例文件/输入文件.xlsx*** ；

3.检核窗口将自动进行计算，如有超限，将会在右上角铃铛处弹出警告；

4.在确保输入无误后，点击导出结果，文件内容格式将如 ***/示例文件/输出文件.xlsx*** 所示；

5.有超限的进行输出后会在特定格子显示；

6.根据各测站高程，用户需自行选取出测点的高程；

7.有任何错误发生时请重新按照1-5的步骤进行，数据将自动清空。

### 注意：
  由于为软件DEMO，作者未制作输入错误或BUG发生的提示窗口，如果进行输入后软件没有任何反应，请点击重置按钮或重启软件后重新输入，并仔细检查输入文件的格式是否符合规范。

### 按照实习要求，具体操作如下：
  在测站数量栏输入23，敲击回车，点击输入文件按钮，弹窗选择文件路径 ***/输入输出示例/输入文件.xlsx*** ，此时左栏显示所有验核数据。右上角未提示超限。点击导出结果并选择路径，可以看到所有高程结果。

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
  各测站内的计算已经由数据库内部的λ表达式自动完成，代码如下：（除改正后平均高差）
 ```
  public class DataDetails
  {
      /// 后视距，待求
      public float PosteriorSightDistance => MathF.Abs(upperThreadOfRearRuler / 10f - lowerThreadOfRearRuler / 10f);

      /// 视距差，待求
      public float Parallax => PosteriorSightDistance - ForwardSightDistance;

      /// 前视距，待求
      public float ForwardSightDistance => MathF.Abs(upperThreadOfFrontRuler / 10f - lowerThreadOfFrontRuler / 10f);

      /// 总视距
      public float TotalSightDistance => ForwardSightDistance + PosteriorSightDistance;

      /// 黑色面后尺减前尺，待求
      public float DifferenceBetweenBlackRearAndFront => ((float)blackLevelReadingOfRearRuler / 1000 - (float)blackLevelReadingOfFrontRuler / 1000);

      /// 红色面后尺减前尺，待求
      public float DifferenceBetweenRedRearAndFront => ((float)redLevelReadingOfRearRuler / 1000 - (float)redLevelReadingOfFrontRuler / 1000);

      /// 后尺黑+K-红，待求
      public int RearLevelCheck => blackLevelReadingOfRearRuler + K - redLevelReadingOfRearRuler;

      /// 前尺黑+K-红，待求
      public int FrontLevelCheck => blackLevelReadingOfFrontRuler + K - redLevelReadingOfFrontRuler;

      /// 后尺-前尺的黑+K-红，待求
      public int RearFrontDifferenceLevelCheck => RearLevelCheck - FrontLevelCheck;

      /// 平均高差，待求
      public float AverageHeightDifference => 0.5f * (DifferenceBetweenBlackRearAndFront + DifferenceBetweenRedRearAndFront);

      /// 改正高差，待求
      public float revisedHeightDifference;

      /// 高程，待求
      public float height;
  }
 ```
  然后由 ***/源文件/Assets/Scrips/DataManager.cs*** 接管这些数据，并做一些必要的加和处理，如计算高差闭合差、视距和、总距离等，并依此得到高差闭合差的距离分配系数，计算得到各测站的改正后高差。检核也在此处完成，先针对各测站的视线长、前后视距离差、红黑面读数差和红黑面高差之差做判断，之后再总体上判断环路闭合差与前后视距离累计差是否符合条件，代码如下所示：
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

#### 5.GUI
  本软件搭载了个性化的GUI操作界面，便于用户查看与修改数值。总体上，UI显示控制由 ***/源文件/Assets/Scrips/UIManager.cs*** 全权控制，并且不会耦合至其他功能类中。控制按钮点按发生事件、在对应格子显示数值皆独立于数据处理存在，实现了分工合作与充分解耦。
  
#### 6.工具类
  ***/源文件/Assets/Scripts/EventHandler.cs*** 全局静态类事件中心，是观察者模式的扩展脚本，其目的是将各功能类充分解耦，增强软件的可扩展性，未来的所有事件都可在此声明。

  ***/源文件/Assets/Scripts/Singleton.cs*** 泛型单例类，一些 ` manager `文件全局仅存在一个，使他们成为单例便于其他类进行访问调用。本软件仅 ***/源文件/Assets/Scripts/DataManager.cs*** 继承了该单例父类，但是为了后续扩展，还是选择将单例写为基类。

-----
  
## 数据概况与实习经验
  
  为检验程序正确性，使用了 ***/输入输出示例/输入文件.xlsx*** 作为输入对象，得到的结果较为可观，无超限数据。结果导出如 ***/输入输出示例/输入文件.xlsx*** 所示，数据自动填写的结果经检验正确。
  
  为了精益求精，我在寻求可视化操作界面的解决方案上花了不少时间。主要难点体现在C#的Windows窗体不好控制数据的传输，最终选择Unity引擎作为开发环境，事实也证明，使用专注于图形处理的引擎更加得心应手，也让我有余力对界面进行一些美化处理。
  
  另外，在如何实现用户自填写数据上，我同样花费了一些心思。但是本次实习填写的数据过多，测站高达23个，在不制作滚动窗口的情况下无法实现按测站数量生成对应数量的测站填写栏供用户手动输入数据，是未来可以新增的功能。
  
  实习过程中，我还发现了不少问题，一些细微的BUG，如对象时不时地报空，可能要花费我一整个上午去纠察。不过看到软件成型的时候，还是不由自主地感到兴奋，这便是我热爱编程的原因。也感谢测量实习给了我一个温故Unity与C#的机会，让我拥有了独自编制软件的宝贵经验。