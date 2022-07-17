# 导线测量
  本软件由 **徐晨昊2054153** 使用 **C#及Unity引擎** 独立开发，仅供测量实习使用。
  本软件可以进行闭合导线的坐标计算。
  
-----

## 使用者文档

  软件位于 ***/软件DEMO.exe*** ，打开软件后，请按下列步骤进行操作：
  
1.在右侧输入栏输入正确的测站数量、起算方位角与起算坐标；

2.输入数据。您可以在左侧的输入框内手动输入，也可以点击导入文件按钮，文件内容格式请参考 ***/输入输出示例/输入文件.xlsx*** ；

3.根据需要进行数据修改，或者点击添加扰动按钮使所有数据产生微小浮动。每一次更改后，程序都将自动进行计算，并将结果显示在右下的信息栏内。如有超限，将会在右上角铃铛处弹出警告，您可以多点按几次添加扰动按钮以验证警告信息的正确性；

4.在确保更改结束后，点击导出结果到指定位置，文件内容格式将如 ***/输入输出示例/输出文件.xlsx*** 所示；

5.输出的Excel文件中，所有对数据的更改都将保留，检核计算与超限提醒将在特定格子显示，用户根据自己的需要截取数据；

6.由于制作仓促，难免有难以发现的错误。遇到错误发生时请点击重置按钮或重启软件，并重新从步骤1开始操作。

-----

## 开发者文档

代码及UI素材源文件位于 ***/源文件*** 中，您可直接打开 ***/源文件/导线测量.Sln*** 参考代码。

### 下面将讲解各脚本的使用方法：（详细参见各脚本注释）

#### 1.数据库
  本软件使用了Unity命名空间下的ScriptableObject类将数据统一存储在Asset资源包中，便于其他脚本统一访问。***/源文件/Assets/DataCollection/DataDetails.cs*** 文件定义了一个测站需要得到的所有数据，并以测站为一个数据块的单位，将这些数据块存储在 ***/源文件/Assets/DataCollection/Data_SO.cs*** 声明的List数据库中。
  另外，由于本次数据含有按度、分、秒输入的角度信息，因此由 ***/源文件/Assets/DataCollection/Angle.cs*** 定义了角的具体字段与方法，在其中重载了加号、减号与乘号运算符，并实现了 `System.IComparable<T>` 接口供后续角度运算使用。

#### 2.文件开闭
  为提高软件复用性，必须实现文件IO的操作。文件操作位于`XCH.SurveyPractice.FileOperation`的命名空间中，代码可以复用至任何需要文件读写的软件里。***/源文件/Assets/DataCollection/OpenFileName.cs*** 定义了读写文件的数据结构；***/源文件/Assets/Scripts/LocalDialog.cs*** 中实现了以系统调用方式呼出“打开文件”与“保存文件”的Windows窗体； ***/源文件/Assets/Scripts/SelectFile.cs*** 则使用这些方法与UI按钮绑定，记录用户读写的位置，便于后面对Excel文件进行打开与保存的操作。
  
#### 3.读写Excel
  本软件使用了EPPlus类库读写Excel表格。在获得用户选择的路径后，***/源文件/Assets/Scripts/ContactWithExcel.cs*** 将根据该路径打开对应的Excel文件，并按既定的格式读取内容至数据库中，与此同时，完成一个测站内部分数据的计算，如两目标点位左盘差、右盘差、转折角。在用户导入完毕即将导出时，该脚本将数据库中的数据重新写回Excel，并可根据用户需要另存为至任意位置。
  
#### 4.数据处理与验核
  由 ***/源文件/Assets/Scrips/DataManager.cs*** 接管从Excel中读取的数据，并做一些必要的加和处理，如计算测站间距离、转折角总和、导线总长度、角度闭合差、方位角、坐标增量、坐标增量闭合差、改正坐标增量、以及最终的结果坐标。`DataManager.Calculate()` 方法实现了这些步骤，由于为程序的核心步骤，将其展示在此：
```
private void Calculate()
    {
        coordDeltaDifference = new Coordinate();
        originalAngleSum = new Angle(0, 0, 0);
        distanceSum = 0;

        //计算各测站与下个测站的距离
        for (int i = 0; i < spotCount; i++)
        {
            DataDetails data = data_SO.dataList[i];
            if (i < spotCount - 1)
                data.distance = (data.aLeftLength + data.aRightLength + data_SO.dataList[i + 1].bLeftLength + data_SO.dataList[i + 1].bRightLength) / 4;
            else
                data.distance = (data.aLeftLength + data.aRightLength + data_SO.dataList[0].bLeftLength + data_SO.dataList[0].bRightLength) / 4;
        }

        foreach (var data in data_SO.dataList)
        {
            //计算转折角总和
            originalAngleSum += data.OriginalAngle;

            //计算导线总长度
            distanceSum += data.distance;
        }

        //计算角度闭合差
        Angle theoryAngleSum = new Angle((spotCount - 2) * 180, 0, 0);
        originalAngleDifference = originalAngleSum - theoryAngleSum;

        for (int i = 0; i < spotCount; i++)
        {
            DataDetails data = data_SO.dataList[i];

            //修正转折角
            data.revisedOriginalAngle = data.OriginalAngle - (1 / (float)spotCount) * originalAngleDifference;

            //计算方位角
            if (i == 0)
                data.posAngle = startPosAngle + halfRound - data.revisedOriginalAngle;
            else
                data.posAngle = data_SO.dataList[i - 1].posAngle + halfRound - data.revisedOriginalAngle;

            while (data.posAngle.CompareTo(fullRound) > 0)
                data.posAngle -= fullRound;

            while (data.posAngle.CompareTo(zero) < 0)
                data.posAngle += fullRound;

            //计算坐标增量
            float radian = (float)(Mathf.PI * (data.posAngle.degree + (float)data.posAngle.minute / 60 + (float)data.posAngle.second / 3600) / 180);

            data.delta = new Coordinate
            {
                x = (data.distance * Mathf.Cos(radian)),
                y = (data.distance * Mathf.Sin(radian))
            };

            //计算坐标增量闭合差
            coordDeltaDifference.x += data.delta.x;
            coordDeltaDifference.y += data.delta.y;
        }

        for (int i = 0; i < spotCount; i++)
        {
            DataDetails data = data_SO.dataList[i];

            //计算改正后坐标增量
            data.revisedDelta = new Coordinate
            {
                x = data.delta.x - coordDeltaDifference.x * data.distance / distanceSum,
                y = data.delta.y - coordDeltaDifference.y * data.distance / distanceSum
            };

            //计算坐标
            if (i == 0)
                data.coordinate = new Coordinate
                {
                    x = startPos.x + data.revisedDelta.x,
                    y = startPos.y + data.revisedDelta.y,
                };
            else
                data.coordinate = new Coordinate
                {
                    x = data_SO.dataList[i - 1].coordinate.x + data.revisedDelta.x,
                    y = data_SO.dataList[i - 1].coordinate.y + data.revisedDelta.y,
                };
        }
    }
```
  
  之后，对计算好的数据进行检核，先针对各测站盘左盘右角度差与最大最小距离差是否超限做判断，之后再总体上判断角度闭合差与相对闭合差是否符合条件，代码如下所示：
 ```
   private void CheckLimit()
    {
        overLimitType = OverLimitType.未超限;

        Angle limitAngle = new Angle(0, 0, 40);
        Angle limitAngleDifference = new Angle(0, 0, (int)(60*Math.Sqrt(spotCount)));

        for (int i = 0; i < spotCount; i++)
        {
            DataDetails data = data_SO.dataList[i];

            //检核盘左盘右角度差超限
            if ((data.OriginalLeftDifferenceAngle - data.OriginalRightDifferenceAngle).CompareTo(limitAngle) > 0 ||
               (data.OriginalLeftDifferenceAngle - data.OriginalRightDifferenceAngle).CompareTo(zero - limitAngle) < 0)
                overLimitType = OverLimitType.盘左盘右角度差超限;

            List<float> length = new List<float>();
            length.Add(data.aLeftLength);
            length.Add(data.aRightLength);
            if (i < spotCount - 1)
            {
                length.Add(data_SO.dataList[i + 1].bLeftLength);
                length.Add(data_SO.dataList[i + 1].bRightLength);
            }
            else
            {
                length.Add(data_SO.dataList[0].bLeftLength);
                length.Add(data_SO.dataList[0].bRightLength);
            }

            //检核最大最小距离差超限
            length.Sort();
            if (length[3] - length[0] > 0.1)
                overLimitType = OverLimitType.最大最小距离差超限;
        }

        //检核角度闭合差超限
        if (originalAngleDifference.CompareTo(limitAngleDifference) > 0 || originalAngleDifference.CompareTo(zero - limitAngleDifference) < 0)
            overLimitType = OverLimitType.角度闭合差超限;

        //检核相对闭合差超限
        if (CompareDifference > 4000)
            overLimitType = OverLimitType.相对闭合差超限;
        
        //呼叫超限事件，进行UI显示
        EventHandler.CallOverLimitEvent(overLimitType);
    }
  ```
  如有超限的数据，***/源文件/Assets/Scrips/DataManager.cs*** 将通过 ***/源文件/Assets/Scripts/EventHandler.cs*** 通知 ***/源文件/Assets/Scripts/UIManager.cs***，后者将在屏幕上弹出超限类型警告，此时输出文件，特定单元格中同样将提示超限。
  
#### 5.GUI
  本软件搭载了个性化的GUI操作界面，便于用户查看与修改数值。总体上，UI显示控制由 ***/源文件/Assets/Scrips/UIManager.cs*** 全权控制，并且不会耦合至其他功能类中。除了控制按钮点按发生事件、在对应格子显示数值以外，该脚本控制着 ***/源文件/Assets/Scrips/SlotUI.cs*** 的子脚本，用于在屏幕左侧生成对应数量的测站信息栏供用户输入。各测站内部的信息显示由每个SlotUI脚本自行控制，从而实现了分工合作与充分解耦。
  
#### 6.工具类
  ***/源文件/Assets/Scripts/EventHandler.cs*** 作为全局静态类，是各事件的事件中心，其目的是将各功能类充分解耦，增强软件的可扩展性，未来的所有事件都可在此声明。
  ***/源文件/Assets/Scripts/Singleton.cs*** 泛型单例类，一些manager文件全局仅存在一个，使他们成为单例便于其他类进行访问调用。本软件仅 ***/源文件/Assets/Scripts/DataManager.cs*** 继承了该单例父类，但是为了后续扩展，还是选择将其写为父类。
