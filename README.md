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

7.如果窗口发生畸变请调整窗口至合适大小比例。

### 注意：
  由于为软件DEMO，作者未制作输入错误或BUG发生的提示窗口，如果进行输入后软件没有任何反应，请点击重置按钮或重启软件后重新输入，并仔细检查输入文件的格式是否符合规范。

### 按照实习要求，具体操作如下：
  在测站数量栏输入4，敲击回车，此时右下栏弹出相应数量的测点栏。点击导入文件按钮，弹窗选择文件路径 ***/输入输出示例/输入文件.xlsx*** ，此时右栏显示所有输入信息，并在左栏显示测站点与后视点的相对位置图像（您也可以手动输入，不过尚有BUG未更改，不建议）。点击测点栏编号左侧的圆形按钮，对应测点的图像将在左栏显示，一并显示的还有平距与平面夹角。此时可以根据需要修改右栏的数据，左栏图像会相应变化。对数据满意时点击导出结果按钮并选择路径，所有手动输入结果都将一并保存。

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
  
  另外，本软件实现了测点坐标的可视化， ***/源文件/Assets/Scrips/UIManager.cs*** 负责在需要的时刻生成测站的图像并作连线，其核心代码如下所示：
  
```
private void OnInstantiateSpotPrefab(Coordinate coord, int id)
    {
        //在对应位置生成测点图片
        Vector3 targetPos = new Vector3((float)coord.x, (float)coord.y, 0);
        RectTransform spotRect = Instantiate(spotPrefab, targetPos, Quaternion.identity, detailsInfo).GetComponent<RectTransform>();
        spotRect.localPosition = targetPos;
        spotRect.GetComponentInChildren<Text>().text = id.ToString();

        //与测站点连线
        CreateLine(Vector3.zero, targetPos);

        //在线段中点生成距离
        Vector2 middlePos = GetBetweenPoint(Vector3.zero, targetPos);
        RectTransform txtRect = Instantiate(textPrefab, middlePos, Quaternion.identity, detailsInfo).GetComponent<RectTransform>();
        txtRect.localPosition = middlePos;
        txtRect.GetComponent<Text>().text = data_SO.dataList[id - 1].horizontalDistance.ToString("#0.000");

        //在测站图片下方生成与后视点的夹角
        RectTransform angleTextRect = Instantiate(textPrefab, Vector3.zero + 80 * Vector3.down, Quaternion.identity, detailsInfo).GetComponent<RectTransform>();
        angleTextRect.localPosition = Vector3.zero + 80 * Vector3.down;
        angleTextRect.GetComponent<Text>().text = 
            data_SO.dataList[id - 1].Degree.ToString() + "° " + data_SO.dataList[id - 1].Minute.ToString() + "' " + data_SO.dataList[id - 1].Second.ToString() + "''";
    }
  ```
  应该注意的是，为使生成的图像元素不随窗口变动而变动，应使用 `RectTransform.localPosition` 字段而非 `RectTransform.position` ，即应使用元素与父物体的相对坐标，这也是作者在进行开发时遇到的主要问题之一。
  
#### 6.工具类
  ***/源文件/Assets/Scripts/EventHandler.cs*** 全局静态类事件中心，是观察者模式的扩展脚本，其目的是将各功能类充分解耦，增强软件的可扩展性，未来的所有事件都可在此声明。
  
  ***/源文件/Assets/Scripts/Singleton.cs*** 泛型单例类，一些 ` manager `文件全局仅存在一个，使他们成为单例便于其他类进行访问调用。本软件仅 ***/源文件/Assets/Scripts/DataManager.cs*** 继承了该单例父类，但是为了后续扩展，还是选择将单例写为基类。
  
  -----
  
  ## 数据概况与实习经验
  
  为检验程序正确性，使用了 ***/输入输出示例/输入文件.xlsx*** 作为输入对象，得到的结果较为可观，输出数据的结果与显示的结果（角度大小、线段长短）一致，可以认为测点的显示结果是正确的。
  
  为了精益求精，我在寻求可视化操作界面的解决方案上花了不少时间。主要难点体现在如何将用户输入的测点坐标体现在图像上，最终采用计算出各测点相对测站的坐标，将测站点置于显示窗口的中心，如此一来便形成了一平面直角坐标系，然后将测站栏UI制作成预制体的方式，交由 `UIManager` 在指定的时机按这个相对坐标生成，可以看到，最终效果还算理想，但是一个主要问题是，由于搭载这些测点栏的父物体不是可调整比例的窗体，一旦几个测点的相对坐标相差较远，坐标图像就会生成到屏幕外面去（不过此时导出的结果依然是正确的）。由于时间不足，我尚未对此做出改进，是未来可以进一步探究的命题。
  
  另外，在如何实现用户自填写数据上，我同样花费了一些心思。如果导入文件后发现数据有误，回到Excel更改数据再重新导入比较麻烦，因此设计了可以直接在GUI上更改数据的方案，即将UI的 `Text` 组件全部更换为 `InputField` ，最终改动的数据将由 `InputField.OnEndEdit()` 方法写回数据库中，在实现改动的保存同时，也保证了UI功能类的独立性。
  
  实习过程中，我还发现了不少问题，一些细微的BUG，如对象时不时地报空，可能要花费我一整个上午去纠察。不过看到软件成型的时候，还是不由自主地感到兴奋，这便是我热爱编程的原因。也感谢测量实习给了我一个温故Unity与C#的机会，让我拥有了独自编制软件的宝贵经验。
