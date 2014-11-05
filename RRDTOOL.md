# RRDTool Usage

##1.如何使用rrdtool创建各种类型、特性的RRD环型数据库。


```bash
rrdtool create filename [--start|-b start time] [--step|-s step] [DS:ds-name:DST:dst arguments] [RRA:CF:cf arguments]
```

说明：

RRDtool的创建功能能够设置一个新的RRD数据库文件。该功能完成所创建的文件全部被预填入 UNKNOWN 数值。

* **filename**   
    需要创建的RRD的文件名。RRD数据库文件名应当以 .rrd作为扩展名。尽管RRDtool可以接受任何文件名。

* **--start|-b start time(default: now - 10s)**  
    设定RRD数据库加入的第一个数据值的时间－从1970-01-01 UTC时间以来的时间（秒数）。RRDtool不会接受早于或在指定时刻上的任何数值。

* **--step|-s step(default: 300 seconds)**   
    指定数据将要被填入RRD数据库的基本的时间间隔（默认是300秒）。

* **DS:ds-name:DST:dst arguments**   
    单个RRD数据库可以接受来自几个数据源的输入。例如某个指定通讯线路上的进流量和出流量。在DS配置选项中，你必须为每个需要在RRD存储的数据源指定一些基本的属性。  
    *ds-name*是你要用来从某个RRD中引用的某个特定的数据源。ds-name必须为[a-zA-Z0-9]间的、长度为1－19个字符组成。  
    DST定义数据源的类型。数据源项的后续参数依赖于数据源的类型。   

    - 对于GAUGE、COUNTER、DERIVE、以及ABSOLUTE，其数据源的格式为：  
    ```
    DS:ds-name:GAUGE | COUNTER | DERIVE | ABSOLUTE:heartbeat:min:max
    ```  

    - 对于COMPUTE数据源，其格式为:  
    ```
    DS:ds-name:COMPUTE:rpn-expression
    ```

    要确定使用哪种数据源类型，请检查下面的定义。
    - **GAUGE**  
    是像温度计、或者某个房间内的人数、或共享Redhat的值这样的东西。

    - **COUNTER**  
    是像路由器中 ifInOctets 计数器这样会持续递增的计数器。COUNTER数据源假设计算机永远不会减小，除非计数器溢出。update功能可能导致溢出。计算机是按照每秒的频率存储的。当计数器溢出时，RRDtool会检查该溢出是否会发生在32位或64位边界，并且相应的把合适的值加入结果中。

    - **DERIVE**  
    存放该数据源从以往到差异线。这对于gauges类型非常有用，它可用来衡量进出某个房间的比率。在derive内部，与COUNTER几乎是一样的，但是没有溢出检查。因此，如果你的计数器在32或64位不会复位，你应当使用DERIVE或者用一个MIN值为0的混合使用。  
如果你不容许偶尔发生的、某个计数器的合法回绕复位而造成的错误，而要用“unknowns”来对表示所有计数器的合法回绕和复位，你就要使用min=0的DERIVE类型。否则，使用具有合理max的COUNTER类型，会为所有的合法计数器回绕返回正确的值。  
对于一个步长为5分钟的32位计数器，计数器回绕复位的错误概率大约为：每1Mbps的最大带宽发生概率为0.8%.注意这等价于100Mbps接口的80%，因此对于高带宽接口和32位计数器，最好使用带有min=0的DERIVE。如果你使用的是64位计数器，只有任何最大值的设定可以避免计数器回绕的错误发生的可能性。  

    - **ABSOLUTE**  
读取后马上复位的计数器。用于易于溢出的快速计数器。因此，不要常规地读取他们，你需要自每次读取后确认在下一次溢出前有一个最大的有效时间。该类型的另外一个用途是你需要累积上次更新以来的信息数目。

    - **COMPUTE**  
用于存放对RRD中的其他数据源进行公式计算的结果。该数据源在更新时不需要提供数值，它是根据rpn－表达式定义的公式从数据源的PDPs中计算出来的PDP（Primary Data Point）。归并功能会被应用到COMPUTE数据源的PDPs上。在数据库软件中，此类数据集用“虚拟”或“计算”列表示。  

    - **heartbeat**  
定义了在两次数据源更新之间、在将数据源的数值确定为 UNKNOWN 前所允许的最大秒数。  

    - **min**和**max**   
    定义了数据源提供、预期的数值范围。任何数据源的超过min或max数值范围的数值，都将被认为是UNKNOWN 。如果你不知道或者不关心mix和max, 将他们设置为 unknown。注意min和max总是值数据源所处理的数值。对于一个流量计数器类型的DS来说，这可以是预期中该设备获取的数据率。  
如果有可用的min/max的值信息，一定要设置min和max属性。这可以帮助RRDtool在更新时对提供的数据进行健壮检查。

    - **rpn－expression**  
定义了由同一个RRD库的其他数据源的计算而来的、某个COMPUTE数据源的PDPs计算公式。这于graph命令的CDEF参数一样。请参看graph手册了解RPN操作符的列表和说明。对于COMPUTE数据源，不支持以下RPN操作符：COUNT、PREV、TIME、和LTIME。此外，在定义RPN表达式时，COMPUTE数据源只能够引用在create命令中列出的数据源。这于CDEF的限制是一样的，CDEF只能够引用在同一个graph命令中前面定义的DEFs和CDEFs。

* **RRA:CF:cf arguments**  
    RRD的一个目的是在一个环型数据归档中存储数据。一个归档有大量的数据值或者每个已定义的数据源的统计，而且它是在一个RRA行中被定义的。  
当一个数据进入RRD数据库时，首先填入到用 -s 选项所定义的步长的时隙中的数据，就成为一个pdp值－首要数据点（Primary Data Point）。  
该数据也会被用该归档的CF归并函数进行处理。可以把各个PDPs通过某个聚合函数进行归并的归并函数有这样几种：AVERAGE、MIN、MAX、LAST等。这些归并函数的RRA命令行格式为:  
    ```
    RRA:AVERAGE | MIN | MAX | LAST:xff:steps:rows
    ```

    - **xff**
xfiles factor定义了在被归并数值仍然是一个未知时，*UNKNOWN*数据中，某个归并间隔的哪个部分可以采用。

    -**steps**
定义这些PDP中的多少个可以用来构建归并的数据点。

    -**rows**
定义在一个RRA归档中保留多少次的生成数据值。

例1
```bash
rrdtool create temperature.rrd --step 300 \
DS:temp:GAUGE:600:-273:5000 \
RRA:AVERAGE:0.5:1:1200 \
RRA:MIN:0.5:12:2400 \
RRA:MAX:0.5:12:2400 \
RRA:AVERAGE:0.5:12:2400
```

上例设置了一个名为 temperature.rrd 的RRD，它每300秒接收一个温度值。如果超过600秒没有提供数据，温度值变为*UNKNOWN*。其最小可接受的值为 -273,最高值为5000.

本例中同时还定义了几个归档区。第一个RRA归档区存储100小时内的温度（1200\*300秒=100小时）。第二个RRA存储每小时的最低温度（12\*300秒=1小时），共存储100天的数据（2400小时）。第三和第四个RRA分别存放最高温度和平均温度。

例2
```bash
rrdtool create proxy.rrd --step 300 \
DS:Total:DERIVE:1800:0:U \
DS:Duration:DERIVE:1800:0:U \
RRA:AVERAGE:0.5:1:2016
```

本例是监视一个Web代理每300秒间隔（5分钟）内处理的请求的平均请求数。此例中，该代理有两个计数器，启动后处理的请求总数、以及处理请求的合计累积数。显然这些计数器都有某个回绕点，但是使用DERIVE数据源类型同时还可以处理在Web代理停止和重启时的复位。

在该RRD数据库中，存储的第一个数据源类型是间隔期内的每秒请求数。第二个数据源类型是在除以300的间隔期内的请求处理总数。

##2.rrd环型数据库的更新

```bash
rrdtool {update | updatev} filename [--template|-t ds-name[:ds-name]...] N|timestamp:value[:value...] at-timestamp@value[:value...] [timestamp:value[:value...] ...]
```

>filename ：要更新的RRD数据库的名称。

>--template|-t ds-name[:ds-name]... ：-t ds-name要更新RRD数据库中数据源的名称

>N|timestamp:value[:value...]：时间：要更新的值...

```bash
timestamp=`date -d "2003/08/15 12:00" +%s`
```

##3.如何绘制rrd环型数据库中的采集到的数据

```bash
rrdtool graph filename [option ...] [data definition ...] [data calculation ...] [variable definition ...] [graph element ...] [print element ...]
```

>filename 要绘制的图片名称

>[-s|--start time] 启始时间[-e|--end time]结束时间 [-S|--step seconds]步长

>[-t|--title string]图片的标题 [-v|--vertical-label string] Y轴说明

>[-w|--width pixels] 显示区的宽度[-h|--height pixels]显示区的高度 [-j|--only-graph]

>[-u|--upper-limit value] Y轴正值高度[-l|--lower-limit value]Y轴负值高度 [-r|--rigid]

>DEF:vname=rrdfile:ds-name:CF[:step=step][:start=time][:end=time]

>CDEF:vname=RPN expression

>VDEF:vname=RPN expression


主要用处是说明您要取出那个RRD档案的 DSN 到这个 graph 的参数中来 CDEF 通过运算得到一个虚拟的变量,,其运算式需写成后序 EX: a=1+3 写成 a=1,3 + LINE{1|2|3}:vname[#rrggbb[:legend]] LINE1:your_var#rgb颜色值:图例说明,这个 "your_var" 需存在 DEF 或 CDEF 的宣告中, AREA:vname[#rrggbb[:legend]] AREA 画出样本数值至 0 之间的区块图 STACK:vname[#rrggbb[:legend]] STACK 叠在上一个值上的图形 请注意,如果使用 AREA/STACK 时需特别注意图盖图的问题,一定要先画大的值, 再画小的值,这才会有层次的效果,不然,最大的数据若最后画,会盖住前面的数据 COMMENT 说明文字,如 COMMENT:"Last Updated" 将在图上产生该文字,可以用 \n 等换行符号 GPRINT GPRINT:vname:CF:format vname 即DEF 中的 your_var,而 CF 看你要输出的文字是 AVERAGE/MAX/MIN/LAST 等数值,format 如同 printf 中的格式, EX: GPRINT:telnet:AVERAGE:"%10.0lf \n" 意即要输出这段时间中 (-s ~ -e 中,telnet的平均值,%10.0lf 则是为了好算位置)。
