---
title: 阅读DDoS攻击检测源码笔记
date:  2020-12-17 16:46:34
category: project
tags: api
excerpt: 利用机器学习检测DDoS攻击源码分析
---


### pandas API:[https://pandas.pydata.org/docs/reference/index.html#api](https://pandas.pydata.org/docs/reference/index.html#api)

+ 1 **pd.read_csv('data.csv'):**返回DataFrame对象
 
	`normal = pd.read_csv('data/csv/normal.csv')`
    
+ 2 **pandas.DataFrame.loc:**使用标签来选择一组数据
 
		attack_http_switch.loc[:,'Source'] = '172.24.1.81'  #选择'Source'这一列所有数据
		normal_switch = normal[normal['Source'] == '172.24.1.81']  #选择normal中'Source'为172...的数据
		attack_http = attack_http[ (attack_http['Source'] == '172.24.1.67') & (attack_http['Destination'] == '172.217.11.36')]
	**PS:.loc[ ]通过索引(0,1,2...)选择数据，.ix[ ]可以通过索引和标签混合选择数据**

+ 3 **pandas.concat:**简单的对DataFrame进行堆叠，并且空值填充为NaN

		concat([obj1,obj2,obj3], axis=0, join='outer', ignore_index=False, keys=None, 
		levels=None,names=None,verify_integrity=False, sort=False, copy=True)

	axis表示堆叠方向，默认为0，如果是在水平方向堆叠，需要指定axis=1<br>
	join表示堆叠方式，默认为outer，只有outer和inner方式，inner表示只对相同列进行堆叠<br>
	ignore_index表示是否保留原对象的index，默认为False保留<br><br>
	`switch_data = pd.concat([normal_switch, attack_http_switch, attack_tcp_switch, attack_udp_switch])  #合并switch设备流量`
	
	**PS:concat可一次实现对多个pandas对象的堆叠，默认对所有列在垂直方向进行堆叠,index为原来各自的索引**

+ 4 **pandas.merge:**实现两个DataFrame对象之间的合并，类似于sql两个表之间的关联查询

		pd.merge(left, right, how='inner', on=None, left_on=None, right_on=None, left_index=False, 
		right_index=False, sort=False, suffixes=('_x', '_y'), copy=True, indicator=False, validate=None)

	left & right：第一个DataFrame和第二个DataFrame对象，merge只能实现两个DataFrame的合并，无法一次实现多个合并<br>
	on：指定参考column，要求两个df必须至少有一个相同的column，默认为None以最多相同的column为参考<br>
	how：合并的方式，默认为inner取参考column的交集，outer取并集保留所有行；outer、left、right中的缺失值都以NaN填充；left按照左边对象为参考进行合并即保留左边的所有行，right按照右边对象为参考进行合并即保留右边所有行<br>
	left_on=None & right_on=None：以上on是在两个df有相同的column的情况下使用，如果两个df没有相同的column，使用left_on和right_on分别指明左边和右边的参考column<br>
	left_index & right_index：指定是否以索引为参考进行合并<br>
	sort：合并结果是否按on指定的参考进行排序<br>
	suffixed：合并后如果有重复column，分别加上什么后缀<br>
		
		#生成新的特征group_features（dataframe对象）并合并
		#group_features:TimeBin(index),[bandwidth,num_dest,delta_num_dest](columns)
		#data:default(index),[No.,Time,...,Label,TimeBin](columns)
		#指定data[TimeBin]列与group_features的index:[TimeBin]合并，合并后的新对象的index取data的index

		data = data.merge(group_features, left_on='TimeBin', right_index=True)  

	**PS:pd.joins是简化的merge方式，默认用法相当于merge的pd.merge(df1,df2,how = 'left',left\_index = True,right\_index = True)**
	
+ 5 **pandas.DataFrame.diff:**减去上n行/列的值

	`DataFrame.diff(periods=1, axis=0)`
	<br>periods:默认为1，即减去上1行/列的值(periods可以为负数，即减去下n行/列的值）
	<br>axis:默认为0,选择减去上n行的值

		#delta_num_dest = 当前时间窗口的不同目的IP数 - 前一个时间窗口的不同目的IP
		group_features['device_timebin_delta_num_dest'] = group_features['device_timebin_num_dest'].diff(periods=1)
		
+ 6 **pandas.DataFrame.fillna:**使用特定的方式填充NA/NaN

	`DataFrame.fillna(value=None, method=None, axis=None, inplace=False, limit=None, downcast=None)`
	<br>values:需要填充的值，输入可以是字典，Series等
	<br>method:{'backfill','ffill'...},填充方式（可以回退到NaN）

		#上述代码执行后，第一行会变成空值（第一行前面没有数据了），将空值用0填充
		group_features['device_timebin_delta_num_dest'] = group_features['device_timebin_delta_num_dest'].fillna(0)

+ 7 **pandas.DataFrame.copy:**复制dataframe对象

	`DataFrame.copy(deep=True)`
	<br>deep:默认为True,表示深层复制，修改deep不会导致原始dataframe改变,修改原始对象也不会导致新的dataframe改变

		#复制data到features
		features = data.copy(deep=True)

+ 8 **pandas.DataFrame.shift:**以特定的方式整体移动数据

	`DataFrame.shift(periods=1, freq=None, axis=0, fill_value=None)`	
	<br>periods:移动的行数/列数（可以是负数，表示上移或左移）
	<br>axis:移动方向，行/列

		#dT = 当前Time - 前3行的Time
		features['dT'] = features['Time'] - features['Time'].shift(3)

----

### sklearn API:[https://scikit-learn.org/stable/modules/classes.html](https://scikit-learn.org/stable/modules/classes.html)

+ 1  **sklearn.model_selection.train_test_split:**划分数据集

	`sklearn.model_selection.train_test_split(*arrays, **options)`
	<br>返回值分别为训练集特征，测试集特征，训练集标签，测试集标签

		#data为特征，labels为标签,test_size表示测试集比例
		x_train, x_test, y_train, y_test = train_test_split(data, labels, test_size=0.15)

+ 2 **训练模型**		

		#x_train,y_train分别是训练集特征和训练集标签
		classifier.fit(x_train,y_train)

+ 3 **使用模型进行预测**

		#x_test为测试集，返回预测值
		y_predict = classifier.predict(x_test)

+ 4 **sklearn.metrics.precision_recall_fscore_support:**对每个标签计算precision,recall,f1(都是numpy.ndarray对象)

		sklearn.metrics.precision_recall_fscore_support(y_true, y_pred, *, beta=1.0, labels=None, 
		pos_label=1,average=None, warn_for=('precision', 'recall', 'f-score'), sample_weight=None, zero_division='warn')

	y_true&y_pred:分别是真实值和预测值<br>	
	labels:可以控制标签的顺序，即precision[0],precision[1]...分别对应哪个标签

		#计算每个标签对应的precision,recall,f1
		precision, recall, f1, _ = precision_recall_fscore_support(y_test, y_predict)

+ 5 **sklearn.metrics.zero_one_loss:**计算错误率

		`sklearn.metrics.zero_one_loss(y_true, y_pred, *, normalize=True, sample_weight=None)`
	normalize:默认为True,返回错误率，为False时返回出错的个数

		#计算错误率
		error = zero_one_loss(y_test, y_predict)

+ 6 **sklearn.metrics.confusion_matrix：**计算混淆矩阵以评估一个分类器的准确率

	`sklearn.metrics.confusion_matrix(y_true, y_pred, *, labels=None, sample_weight=None, normalize=None)`
	<br>labels:控制对应的标签顺序

		#计算混淆矩阵
		cm = confusion_matrix(y_test, y_predict)

----

### pyplot API:[https://matplotlib.org/2.1.1/api/pyplot_summary.html](https://matplotlib.org/2.1.1/api/pyplot_summary.html)

+ 1 `plt.imshow(cm, interpolation='nearest', cmap=plt.cm.Blues)`

	interpolation='nearest'表示使用最近邻插值法进行图片缩放[8]

+ 2 `plt.colorbar()`
	
	给图片添加颜色条/渐变色条[6]

+ 3 `plt.xticks(tick_marks, classes, rotation=45)`

	这里tick_marks = [0,1]，表示刻度<br>
	这里classes = ['Normal','Attack']，表示刻度标签
	rotation=45:表示标签倾斜45度

+ 4 `thresh = cm.max() / 2.`

	cm.max()表示混淆矩阵中的最大值

+ 5 `for i, j in itertools.product(range(cm.shape[0]), range(cm.shape[1])):`

	product(x,y)生成笛卡尔积
	假设cm.shape = [3,3]，则range(3) = [0,1,2]<br>
	product(3,3) = [[0,0],[0,1],[0,2],[1,0],[1,1],[1,2],[2,0],[2,1],[2,2]]，则每次循环，i&j分别为(0,0),(0,1)...

+ 6 `plt.text(j, i, cm[i, j],horizontalalignment="center",color="white" if cm[i, j] > thresh else "black")`

	在（j,i）的位置添加cm[i,j]标注，水平对齐方式为'center'，如果cm[i,j] > thresh，颜色为白色，否则为黑色[2]

+ 7 `plt.tight_layout()`
	
	自动调整子图参数[7]

+ 8 `ax.spines["top"].set_visible(False)`

	将子图的上边框（边界）设置为不可见

+ 9 `ax.get_xaxis().tick_bottom() `

	应该修改为：`ax.xaxis.tick_bottom()`
	<br>将x轴的刻度以及刻度标签移到最下面

+ 10 **matplotlib.pyplot.tick_params**(axis='both', **kwargs)

		plt.tick_params(axis="both", which="both", bottom="off", top="off", labelbottom="on",
		left="off", right="off", labelleft="on")

	修改为：`plt.tick_params(axis="both", which="both", bottom=False, top=False, left=False, right=False)  #去掉刻度`
	<br>**bottom,top,left,right:** bool or {'on','off'}，控制是否将 **刻度** 标在上下左右边界上
	<br>**labelbottom,labeltop,labelleft,labelright:** bool or {'on','off'}，控制是否将**刻度标签**标在上下左右边界

+ 11 `values_a, base_a = np.histogram(attack_data, bins=1000)`

	bins:表示划分的区间个数，区间范围默认根据attack\_data中的数据的范围<br>
	values\_a表示落在每个区间中的数的个数<br>
	base\_a表示区间，例如对[0,1]划分为10个区间，则base_a = [0,0.1,0.2,...,1.0]

+ 12 `cumulative_a = np.cumsum(values_a)`

	对values_a累加，例如[1,2,4,6] -> [1,3,7,13]<br><br>
	`cumulative_a = cumulative_a/max(cumulative_a)  #除以总个数即为概率`


+ 13 `attack_line = plt.plot(base_a[:-1], cumulative_a, c='blue', label="Attack Traffic")`

	base\_a[:-1]表示取除最后一个元素之外的所有元素

+ 14 `plt.xlim([0,1500])  #设置x轴区间为[0,1500]`

+ 15 `plt.title('Packet Size CDF', fontsize=16, y=1.08)  #y=1.08表示标题靠上，默认为1`

+ 16 `plt.legend(handles=[normal_line[0], attack_line[0]], loc=4, frameon=False, fontsize=16)`

	设置图例，loc=4表示图例在右下，frameon=False去掉图例边框

----

### References

1. [np.set_printoptions()——控制输出方式](https://blog.csdn.net/weixin_43584807/article/details/103093874)
2. [python画图时给图中的点加标签之plt.text](https://blog.csdn.net/zengbowengood/article/details/104324293)
3. [逻辑回归-信用卡欺诈检测代码调试问题（二分类）](https://blog.csdn.net/yuyue_chn/article/details/103173369)
4. [如何在机器学习中使用交叉验证（实例）](https://zhuanlan.zhihu.com/p/128984157)
5. [模型选择之交叉验证](https://zhuanlan.zhihu.com/p/32627500)
6. [Matplotlib：给子图添加colorbar（颜色条或渐变色条）](https://www.jianshu.com/p/d97c1d2e274f)
7. [plt.tight_layout()](https://blog.csdn.net/du_shuang/article/details/84139716)
8. [数字图像处理笔记二 - 图片缩放](https://blog.csdn.net/haluoluo211/article/details/80918147)
9. [pandas之DataFrame合并merge](https://www.cnblogs.com/Forever77/p/11285302.html)