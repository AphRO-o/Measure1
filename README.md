# 放样测量
  本软件由 **徐晨昊2054153** 使用 **C#及Unity引擎** 独立开发，仅供测量实习使用。
  
  本软件可以用于极坐标放样的距离角度计算。
  
-----

## 使用者文档

  软件位于 ***/放样测量DEMO.exe*** ，打开软件后，请按下列步骤进行操作：
  
1.在右侧上部输入栏输入正确的测站数量，此时右侧下部将弹出相应数量的测点坐标输入框；

2.输入数据。您可以点击导入文件按钮直接导入，文件内容格式请参考 ***/输入输出示例/输入文件.xlsx*** ；也可以在输入框内手动输入（此处尚有BUG未解决，不建议）。

3.根据需要进行数据修改。每一次更改后，程序都将自动进行计算调整，并以测站点为原点，在左侧图像栏建立坐标系；

4.在确保更改结束后，点击各个测点坐标栏的圆形按钮，将在图像栏显示出各自的坐标位置、与测站点的距离、以及与后视点的夹角大小。 您此时仍然可以随意修改数据，坐标位置将随您的输入数值改动。您可以点击“清屏”按钮清除已有的测点显示，以防止多个测点信息叠加在一起，也可以点击“重置”按钮清除所有内容；

5.对左栏显示的结果确认无误后，点击“导出结果”按钮，将数据到指定位置，文件内容格式将如 ***/输入输出示例/输出文件.xlsx*** 所示；

5.输出的Excel文件中，所有对数据的更改都将保留，用户根据自己的需要截取数据；

6.由于制作仓促，难免有难以发现的错误。遇到错误发生时请点击重置按钮或重启软件，并重新从步骤1开始操作。

-----

## 开发者文档

代码及UI素材源文件位于 ***/源文件*** 中，您可直接打开 ***/源文件/放样测量.Sln*** 参考代码。

### 下面将讲解各脚本的使用方法：（详细参见各脚本注释）

#### 1.数据库
  本软件使用了Unity命名空间下的ScriptableObject类将数据统一存储在Asset资源包中，便于其他脚本统一访问。***/源文件/Assets/DataCollection/DataDetails.cs*** 文件定义了一个测站需要得到的所有数据，并以测站为一个数据块的单位，将这些数据块存储在 ***/源文件/Assets/DataCollection/Data_SO.cs*** 声明的List数据库中。
  
  另外，由于本次数据含有按X、Y输入的坐标信息，因此由 ***/源文件/Assets/DataCollection/Coordinate.cs*** 定义了坐标的具体字段与方法，在其中重载了加号、减号与乘号运算符供后续坐标运算使用。

#### 2.文件开闭
  为提高软件复用性，必须实现文件IO的操作。文件操作位于`XCH.SurveyPractice.FileOperation`的命名空间中，代码可以复用至任何需要文件读写的软件里。***/源文件/Assets/DataCollection/OpenFileName.cs*** 定义了读写文件的数据结构；***/源文件/Assets/Scripts/LocalDialog.cs*** 中实现了以系统调用方式呼出“打开文件”与“保存文件”的Windows窗体； ***/源文件/Assets/Scripts/SelectFile.cs*** 则使用这些方法与UI按钮绑定，记录用户读写的位置，便于后面对Excel文件进行打开与保存的操作。
  
#### 3.读写Excel
  本软件使用了EPPlus类库读写Excel表格。在获得用户选择的路径后，***/源文件/Assets/Scripts/ContactWithExcel.cs*** 将根据该路径打开对应的Excel文件，并按既定的格式读取内容至数据库中，与此同时，完成一个测站内部分数据的计算，如两目标点位左盘差、右盘差、转折角。在用户导入完毕即将导出时，该脚本将数据库中的数据重新写回Excel，并可根据用户需要另存为至任意位置。
  
#### 4.数据处理与验核
  由 ***/源文件/Assets/Scrips/DataManager.cs*** 接管从Excel中读取的数据，并做一些必要的加和处理，如计算距离与角度。`DataManager.Calculate()` 方法实现了这些步骤，由于为程序的核心步骤，将其展示在此：
  
  
```
private void Calculate()
    {
        //计算后视点的相对坐标
        Coordinate backCompCoord = x * (backPos - startPos);
        backV3 = new Vector3((float)backCompCoord.x, (float)backCompCoord.y, 0);

        for(int i = 0; i < spotCount; i++)
        {
            //计算测点相对坐标
            data_SO.dataList[i].compCoord = x * (data_SO.dataList[i].coordinate - startPos);

            //计算测点原始角
            data_SO.dataList[i].originalAngle = Math.Atan((data_SO.dataList[i].coordinate.x - startPos.x) / (data_SO.dataList[i].coordinate.y - startPos.y));

            //计算测点水平角
            data_SO.dataList[i].horizontalAngle = Math.Abs(data_SO.dataList[i].originalAngle - Alpha);

            //计算测点水平距离
            data_SO.dataList[i].horizontalDistance = Math.Sqrt((data_SO.dataList[i].coordinate - startPos).x * (data_SO.dataList[i].coordinate - startPos).x +
                                                               (data_SO.dataList[i].coordinate - startPos).y * (data_SO.dataList[i].coordinate - startPos).y);
        }
    }
```


#### 5.GUI
  本软件搭载了个性化的GUI操作界面，便于用户查看与修改数值。总体上，UI显示控制由 ***/源文件/Assets/Scrips/UIManager.cs*** 全权控制，并且不会耦合至其他功能类中。除了控制按钮点按发生事件、在对应格子显示数值以外，该脚本控制着 ***/源文件/Assets/Scrips/SlotUI.cs*** 的子脚本，用于在屏幕左侧生成对应数量的测站信息栏供用户输入。各测站内部的信息显示由每个 `SlotUI` 脚本自行控制，从而实现了分工合作与充分解耦。
  
  另外，本软件实现了测点坐标的可视化， ***/源文件/Assets/Scrips/UIManager.cs*** 
  
#### 6.工具类
  ***/源文件/Assets/Scripts/EventHandler.cs*** 全局静态类事件中心，是观察者模式的扩展脚本，其目的是将各功能类充分解耦，增强软件的可扩展性，未来的所有事件都可在此声明。
  
  ***/源文件/Assets/Scripts/Singleton.cs*** 泛型单例类，一些 ` manager `文件全局仅存在一个，使他们成为单例便于其他类进行访问调用。本软件仅 ***/源文件/Assets/Scripts/DataManager.cs*** 继承了该单例父类，但是为了后续扩展，还是选择将单例写为基类。
  
  -----
  
  ## 数据概况与实习经验
  
  为检验程序正确性，使用了 ***/输入输出示例/输入文件.xlsx*** 作为输入对象，得到的结果较为可观，A坐标的计算结果与给出的起算坐标一致，方位角的计算结果与起算方位角相差2秒，无超限数据。结果导出如 ***/输入输出示例/输入文件.xlsx*** 所示，数据自动填写的结果经检验正确。
  
  为了精益求精，我在寻求可视化操作界面的解决方案上花了不少时间。主要难点体现在如何根据用户输入的测站数量弹出对应数量的测站栏供用户填写，最终采用将测站栏UI制作成预制体的方式，交由 `UIManager` 在指定的时机与位置生成，可以看到，最终效果还算理想，但是一个主要问题是，由于搭载这些测站栏的父物体不是可滚动的窗体，一旦测站数量过多，就难以看清并填写数据了（不过此时导出的结果依然是正确的）。由于时间不足，我尚未对此做出改进，是未来可以进一步探究的命题。
  
  另外，在如何实现用户自填写数据上，我同样花费了一些心思。如果导入文件后发现数据有误，回到Excel更改数据再重新导入比较麻烦，因此设计了可以直接在GUI上更改数据的方案，即将UI的 `Text` 组件全部更换为 `InputField` ，该改动为我做随机扰动如何保存这一问题做了解答。随机扰动作为实习的要求之一，尽管不合理，却有实现的必要，在我的软件语境中，要求UI部分与数据处理部分的强交互，稍有不慎便会耦合在一起。最终方案是做一按钮，需要扰动时点击即可，最终改动的数据将由 `InputField.OnEndEdit()` 方法写回数据库中，在实现改动的保存同时，也保证了UI功能类的独立性。
  
  实习过程中，我还发现了不少问题，一些细微的BUG，如对象时不时地报空，可能要花费我一整个上午去纠察。不过看到软件成型的时候，还是不由自主地感到兴奋，这便是我热爱编程的原因。也感谢测量实习给了我一个温故Unity与C#的机会，让我拥有了独自编制软件的宝贵经验。
