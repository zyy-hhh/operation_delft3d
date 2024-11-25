# TMD的使用（制作开边界）

1.在input.txt文件输入开边界的经纬度

![image-20231204194619222](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204194619222.png)

2.运行TMD.m

![image-20231204194816495](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204194816495.png)

3.参数选择如下图

![image-20231204195031723](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204195031723.png)

xxxxxxxxxx RUNTIMEPATH\v93\runtime\win64添加环境变量

# 网页提取水深.xyz文件

1.首先水深文件格式如下

![image-20231204200637059](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204200637059.png)

2.常用下列第二个网页

https://download.gebco.net/

https://www.ncei.noaa.gov/maps/grid-extract/

3.选择ETOPO_2022(Bedrock; 15 arcseconds)

![image-20231204200744514](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204200744514.png)

4.下载数据

得到exportImage.tiff格式的文件，于是导入ArcGis

5.ArcGis相关操作

![image-20231204201058196](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204201058196.png)

![image-20231204201256278](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231204201256278.png)

别忘了点右下角的“运行”！！！

6.得到.txt，将其转化为XYZ文件格式

代码文件water_xyz.m

具体内容如下

```matlab
%网页上下载下来的水深文件.tiff转化为.txt
%将此.txt导入此程序可以将格式转换成XYZ水深文件格式
% 打开文件
filename = 'maoweihai_xyz.txt'; % 更改为实际文件路径
fileID = fopen(filename, 'r');

% 逐行读取并解析网格信息
ncols_str = fgetl(fileID);
ncols = sscanf(ncols_str, 'ncols %d');

nrows_str = fgetl(fileID);
nrows = sscanf(nrows_str, 'nrows %d');

xllcorner_str = fgetl(fileID);
xllcorner = sscanf(xllcorner_str, 'xllcorner %f');

yllcorner_str = fgetl(fileID);
yllcorner = sscanf(yllcorner_str, 'yllcorner %f');

cellsize_str = fgetl(fileID);
cellsize = sscanf(cellsize_str, 'cellsize %f');


% 跳过 NODATA_value 行
fgetl(fileID);

% 读取水深数据
data = zeros(nrows, ncols);
for i = 1:nrows
    line = fgetl(fileID);
    if ~isempty(line)
        nums = str2num(line);
        if length(nums) == ncols
            data(i, :) = nums;
        else
            fclose(fileID);
            error(['数据格式不正确或列数不匹配在行 ' num2str(i) ': ' line]);
        end
    else
        fclose(fileID);
        error(['遇到空行在行 ' num2str(i)]);
    end
end
% 关闭文件
fclose(fileID);

% 打印网格信息以检查
fprintf('ncols: %d\n', ncols);
fprintf('nrows: %d\n', nrows);
fprintf('xllcorner: %f\n', xllcorner);
fprintf('yllcorner: %f\n', yllcorner);
fprintf('cellsize: %f\n', cellsize);

% 初始化存储经纬度和水深的矩阵
result = zeros(nrows*ncols, 3);

% 遍历网格以计算每个单元格的经纬度
for row = 1:nrows
    for col = 1:ncols
        longitude = xllcorner + (col - 1) * cellsize;
        latitude = yllcorner + (nrows - row) * cellsize;
        depth = data(row, col);

        index = (row - 1) * ncols + col;
        result(index, :) = [longitude, latitude, depth];
    end
end

% 输出到新文件
new_filename = 'processed_maoweihai_data.txt';
fileID = fopen(new_filename, 'w');
for i = 1:size(result, 1)
    fprintf(fileID, '%12.8f   %12.8f   %12.8f\n', result(i, 1), result(i, 2), result(i, 3));
end
fclose(fileID);

```

# 补充

1.notepad++选中想要的相近的几列（用鼠标点击起始位置=> 按住ALT和SHIFT=> 用鼠标点击结束位置）

2.ArcGis先右键属性表，再导出表，否则**计算几何**是不能点的

3.注意.bnd文件里面M和N的增减1问题![image-20231207214457304](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231207214457304.png)

4.latlon  or   m水深和网格的对应关系（？？？忘了什么意思了）

5.getdata操作网址[数据提取神器GetData使用教程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/378045402)

getdata网页在线版https://apps.automeris.io/wpd/index.zh_CN.html（适合日期格式）

6.tecplot标签跟随时间变化公式

采用YY/MM/DD格式：&(zonename[activeoffset=1])

**采用数字格式：&(SOLUTIONTIME%dddd dd-mmm-yyyy at hh:mm:ss am/pm)**

**config_d_hydro.xml**



# 嵌套模型

原理：

1.首先准备大模型小模型的基本建模文件，比如大小网格的水深文件，大模型的边界，小模型的边界

2.首先，通过系列步骤得到.obs文件，也就是将小模型的边界位置对应到大模型中（得到××）[这里只是位置信息的对应]

3.然后通过大模型的边界驱动，计算整个大模型的各个物理量分布，相当于我们特意关注.obs位置处的各物理量变化（比如水位）

4.这里的，也就是通过大模型计算出来的.obs的几个水位信息，构成小模型的边界驱动条件，然后进行小模型的计算

 操作流程

![image-20231207211004184](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208113958363.png)

![image-20231208114152131](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208114152131.png)

1.绘制big.grd 和 small.grd  （查看网格质量），big.dep  small.dep

查看网格质量操作如下

![image-20240108134353446](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240108134353446.png)

2.small.bnd（只是一个点线位置关系，与TMD无关，.bca才是与调和常数有关的包含信息的文件）

3.在tools-nesting1输入上述文件，得到2个输出。其中.obs文件包含小模型边界的水位信息（.bct相当于直接给出了水位随时间的变化情况，也就是相当于通过调和常数得到的.bca文件的多个波的叠加态）注：.bcc文件是和物质输运有关的文件，比如![image-20231208114445992](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208114445992.png)

![image-20231208114526596](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208114526596.png)

4.注意格式的对应关系，.bnd<->.bca    .bnd<->.bct

# tecplot

1.去边框操作

![image-20231208154547744](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208154547744.png)

2.速度合成

![image-20231208160000679](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208160000679.png)
$$
{velocity}=sqrt(v5**2+v6**2+v7**2)
$$
![image-20231208160035317](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231208160035317.png)

# 潮流椭圆

L0=(6n-3)，其中L0是中央子午线，n是带号，18带的中央子午线是105，19带的中央子午线是111，20带的中央子午线是117。

![image-20231220151746446](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231220151746446.png)

![image-20231220175621437](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20231220175621437.png)

添加路径的方法如下

![1705666380186](E:\Documents\WeChat Files\WeChat Files\wxid_7ra1t4uct7th22\FileStorage\Temp\1705666380186.png)

潮流椭圆的程序E:\Delft3d\Tide_Ellipsel_figure\Tide_Ellipsel_figure\TideElips.m

# DDBoundaries的使用

要求：网格必须具有相同的类型（因此，都在球面坐标中，或都在笛卡尔坐标中）。网格方向应该相同（在同一方向增加M和N编号）。列与行之间没有耦合，反之亦然。子域接口应为直线（无阶梯接口）。

![image-20240119201103048](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240119201103048.png)

使用方法：

1.画好大小网格，Edit-DD Boundaries-new,构建DDB

2.Operations-Attach Grids at BBD-Regular Grids

3.Operations-Compile DDB-存储.ddb编译文件

4.配置两个（大小）网格文件的mdf文件

5.修改.ddb文件内容，将.grd改为.mdf

5.startDD选择.ddb文件运行

# mdf配置

1.边界，上游一般设置流量/流速Current；下游一般设置水位Water level

# RGFGRID样条线绘制网格

7.fit grid boundary to land boundary(官方）

edit—line—line to land boundary，两个点选中样条线，再点岸线

8.orthogonalise grid

operations—grid properties—orthogonality

operations—orthogonalise grid

9.turning off grid properties

view—grid property—no grid property

10.deleting grid cells

edit—block—delete interior，左击四个点确定矩形范围后右击

![image-20240303223609130](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240303223609130.png)

# Domain

dry points是网格，文件后缀名.dry

thin dams是网格边长，文件后缀名.thd

# operations

discharges 关于后缀名

discharge definitions后缀名.src

discharge data 后缀名,dis

# monitoring

drogue浮标   .par文件

cross-sections截面  .crs文件

# 坐标系

Cartesian Coordinates
Spherical Coordinates

# &(SOLUTIONTIME%dddd dd-mmm-yyyy at hh:mm:ss am/pm)



# 验证  dbstop if error

原始mn

龙门![image-20240827151554577](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240827151554577.png)![image-20240827151625316](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240827151625316.png)![image-20240827134617626](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240827134617626.png)

龙门74_96

钦南47_387

尖山涠123_38

大榄江184_167



翻转后mn![image-20240827134636735](C:\Users\zyy\AppData\Roaming\Typora\typora-user-images\image-20240827134636735.png)

龙门ocean_6   (30,74)

钦州闸下river_1  (59,34)   (59,29)   (59,26)

钦州river_5    精准定位(87,17)  移动后(87,26)/(87,32)

大榄江ocean_1   精准定位(26,64)

钦南river_8 精准定位(43,2)  移动后(43,23)

尖山涠 jianshanwei  精准(44,123)  移动后(41,123)

进喜园 108.633613  21.970652

听音广场  108.632135   21.958374

