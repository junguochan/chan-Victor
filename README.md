# chan-Victor
维克多 缠论
import pandas as pd
from datetime import timedelta, date

def initialize(context):
    g.security = ['510050.XSHG']
    set_universe(g.security)
    set_benchmark('510050.XSHG')#设置基准
    #设置交易费
    set_order_cost(OrderCost(open_tax=0, close_tax=0.001, open_commission=0.0003, close_commission=0.0003, close_today_commission=0, min_commission=5), type='stock')
    #设置滑点
    set_slippage(FixedSlippage(0.002))#如果您没有调用 set_slippage 函数, 系统默认 PriceRelatedSlippage(0.00246)
    g.n = 30 #获取几分钟k线
    ## 获取前几日的趋势
    ''' temp_data 包含处理后最后一个k线的数据
        zoushi 包含关系处理中关于走势的记录
        after_baohan 合并后的k线'''
    g.temp_data, g.zoushi, g.after_baohan = k_initialization(security=g.security[0],num = 10*48,n=g.n)#运行k_initialization函数得到前面历史数据的状态
    ## 每日运行一次"根据跌幅止损"、"根据大盘跌幅止损"
    #run_daily(dapan_stoploss) #根据大盘止损，如不想加入大盘止损，注释此句即可
    # run_daily(sell) # 根据跌幅卖出
    # run_daily(sell2) # 按天亏损率止损 & 固定止盈

def handle_data(context, data):
    security = g.security
    Cash = context.portfolio.cash#portfolio账户当前的资金，标的信息，即所有标的操作仓位的信息汇总，cash已过时等价于 available_cash可用资金, 可用来购买证券的资金

    hour = context.current_dt.hour #获得当前回测相关时间
    minute = context.current_dt.minute
    n = g.n
    if (hour==9 and minute ==30) or (hour==13 and minute ==00):
        pass
    else:
        if minute%n==0: #%求余运算
            print ('time: %s:%s' %(hour,minute))
            # 获得前n分钟的k线
            x = '%sm'%n
            #获取历史数据
            temp_hist = attribute_history(security, 1, str(x),['high', 'low'], df=False)#获取最新k线的最高价和最低价
            # 包含关系处理
            Input_k = {'high':[],'low':[]}
            Input_k['high'].append(g.temp_data['high'])#在一个list里面尾部插入一个数据，temp_data是包含处理后最后一个k线的数据
            Input_k['low'].append(g.temp_data['low'])
            Input_k['high'].append(temp_hist['high'][0])#temp_hist['high'][0]历史数据里面的最高价的第一个数据
            Input_k['low'].append(temp_hist['low'][0]) 
            g.temp_data, g.zoushi, g.after_baohan = recognition_baohan(Input_k, g.zoushi, g.after_baohan) #按位置对应

            # 分型
            fenxing_type, fenxing_time, fenxing_plot, fenxing_data = recognition_fenxing(g.after_baohan)
            '''
            fenxing_type 记录分型点的类型，1为顶分型，-1为底分型
            fenxing_time 记录分型点的时间
            fenxing_plot 记录点的数值，为顶分型去high值，为底分型去low值
            fenxing_data 分型点的DataFrame值（系列数据）
            '''
            # print fenxing_type, fenxing_time, fenxing_plot, fenxing_data
            # 判断趋势是否反转，并买进
            if len(fenxing_type)>7:
                if fenxing_type[0] == -1:#最新的一个分形是底分型（因为是从后往回找的）
                    location_1 = [i for i,a in enumerate(fenxing_type) if a==1] # 找出1在列表中的所有位置，enumerate 函数用于遍历序列中的元素以及它们的下标
                    location_2 = [i for i,a in enumerate(fenxing_type) if a==-1] # 找出-1在列表中的所有位置
                    # 线段破坏
                    case1 = fenxing_data['low'][location_2[0]] > fenxing_data['low'][location_2[1]] 
                    # 线段形成
                    case2 = fenxing_data['high'][location_1[1]] < fenxing_data['high'][location_1[2]] < fenxing_data['high'][location_1[3]]
                    case3 = fenxing_data['low'][location_2[1]] < fenxing_data['low'][location_2[2]] < fenxing_data['low'][location_2[3]]
                    # 第i笔中底比第i+2笔顶高(辅助判断条件，根据实测加入之后条件太苛刻，很难有买入机会)
                    case4 = fenxing_data['low'][location_2[1]] > fenxing_data['high'][location_1[3]]
                    if case1 and case2 and case3 :
                        # 买入
                        order_value(security[0],Cash)
    
            # 每分钟亏损率止损 & 固定止盈
            if len(context.portfolio.positions) > 0:#context.portfolio.positions是账号多头头寸
                for stock in list(context.portfolio.positions.keys()):
                    price = data[stock].pre_close
                    avg_cost = context.portfolio.positions[stock].avg_cost
                    # if (price-avg_cost)/avg_cost >= 0.2 :
                    #     order_target(stock, 0)
                    if (price-avg_cost)/avg_cost <= -0.1 :
                        order_target(stock, 0)
                    elif (price-avg_cost)/avg_cost >= 0.18 :
                        order_target(stock, 0)

            
'''下面为各类函数'''
################################################################

def sell2(context):
    ## 亏损率止损 & 固定止盈
    if len(context.portfolio.positions) > 0: #len（）返回字符串长度
        for stock in list(context.portfolio.positions.keys()):#keys？？
            hist = attribute_history(stock, 1, '1d', 'close',df=False)
            price = hist['close'][0]
            avg_cost = context.portfolio.positions[stock].avg_cost #avg_cost是开仓均价
            if (price-avg_cost)/avg_cost <= -0.1 : #下跌10%止损
                order_target(stock, 0)
            elif (price-avg_cost)/avg_cost >= 0.2 : #上涨20%止盈
                order_target(stock, 0)

def dapan_stoploss(context):
    ## 根据局大盘止损，具体用法详见dp_stoploss函数说明
    stoploss = dp_stoploss(kernel=2, n=10, zs=0.05)
    if stoploss:
        if len(context.portfolio.positions)>0:
            for stock in list(context.portfolio.positions.keys()):
                order_target(stock, 0)
        # return
        
def sell(context):
    # 根据跌幅卖出
    if len(context.portfolio.positions)>0:
        for stock in list(context.portfolio.positions.keys()):
            hist = attribute_history(stock, 3, '1d', 'close',df=False)#获取前三天的收盘价
            if ((1-float(hist['close'][-1]/hist['close'][0])) >= 0.15):#T-1日相对于T-3日跌幅超过15%
                order_target(stock,0)

def recognition_fenxing(after_baohan):
    '''
    从后往前找
    返回值：
    fenxing_type 记录分型点的类型，1为顶分型，-1为底分型
    fenxing_time 记录分型点的时间
    fenxing_plot 记录点的数值，为顶分型去high值，为底分型去low值
    fenxing_data 分型点的DataFrame值
    after_baohan是合并后每个zoushi（k线）点对应的最高价和最低价
    '''
    ## 找出顶和底
    temp_num = 0 #上一个顶或底的位置
    temp_high = 0 #上一个顶的high值
    temp_low = 0 #上一个底的low值
    temp_type = 0 # 上一个记录位置的类型
    end = len(after_baohan['high']) #返回字符串长度，有多少个最高价数据
    i = end-2#合并k线个数-2
    fenxing_type = [] # 记录分型点的类型，1为顶分型，-1为底分型
    fenxing_time = [] # 记录分型点的时间
    fenxing_plot = [] # 记录点的数值，为顶分型去high值，为底分型去low值
    fenxing_data = {'high':[],'low':[]} # 分型点的DataFrame值
    while (i >= 1):
        if len(fenxing_type)>8:#分型点的类型的个数
            break
        else:
            case1 = after_baohan['high'][i-1]<after_baohan['high'][i] and after_baohan['high'][i]>after_baohan['high'][i+1] #i是顶分型，i+1=end-1
            case2 = after_baohan['low'][i-1]>after_baohan['low'][i] and after_baohan['low'][i]<after_baohan['low'][i+1] #底分型
            if case1:
                if temp_type == 1: # 如果上一个分型为顶分型，则进行比较，选取高点更高的分型 
                    if after_baohan['high'][i] <= temp_high:#i时刻的最高价小于i+1时刻的最高价
                        i -= 1#进行下一个比较
                    else:#i时刻的最高价大于i+1时刻的最高价
                        temp_high = after_baohan['high'][i]#i时刻最高价记入temp_high?
                        temp_num = i#记录这个顶的位置
                        temp_type = 1#记录这是个顶分型
                        i -= 4#?
                elif temp_type == 2: # 如果上一个分型为底分型，则记录上一个分型，用当前分型与后面的分型比较，选取同向更极端的分型
                    if temp_low >= after_baohan['high'][i]: # 如果上一个底分型的底比当前顶分型的顶高，则跳过当前顶分型。
                        i -= 1
                    else:
                        fenxing_type.append(-1)
                        # fenxing_time.append(after_baohan.index[temp_num].strftime("%Y-%m-%d %H:%M:%S"))
                        fenxing_data['high'].append(after_baohan['high'][temp_num])
                        fenxing_data['low'].append(after_baohan['low'][temp_num])
                        fenxing_plot.append(after_baohan['high'][i])
                        temp_high = after_baohan['high'][i]
                        temp_num = i
                        temp_type = 1
                        i -= 4
                else:#case1，i时刻是顶分型最开始的时候这样处理
                    if (after_baohan['low'][i-2]>after_baohan['low'][i-1] and after_baohan['low'][i-1]<after_baohan['low'][i]):#i-1是底分型
                        temp_low = after_baohan['low'][i]
                        temp_num = i-1
                        temp_type = 2
                        i -= 4
                    else:
                        temp_high = after_baohan['high'][i]
                        temp_num = i
                        temp_type = 1
                        i -= 4
                    
            elif case2:
                if temp_type == 2: # 如果上一个分型为底分型，则进行比较，选取低点更低的分型 
                    if after_baohan['low'][i] >= temp_low:
                        i -= 1
                    else:
                        temp_low = after_baohan['low'][i]
                        temp_num = i
                        temp_type = 2
                        i -= 4
                elif temp_type == 1: # 如果上一个分型为顶分型，则记录上一个分型，用当前分型与后面的分型比较，选取同向更极端的分型
                    if temp_high <= after_baohan['low'][i]: # 如果上一个顶分型的底比当前底分型的底低，则跳过当前底分型。
                        i -= 1
                    else:
                        fenxing_type.append(1)
                        # fenxing_time.append(after_baohan.index[temp_num].strftime("%Y-%m-%d %H:%M:%S"))
                        fenxing_data['high'].append(after_baohan['high'][temp_num])
                        fenxing_data['low'].append(after_baohan['low'][temp_num])
                        fenxing_plot.append(after_baohan['low'][i])
                        temp_low = after_baohan['low'][i]
                        temp_num = i
                        temp_type = 2
                        i -= 4
                else:
                    if (after_baohan['high'][i-2]<after_baohan['high'][i-1] and after_baohan['high'][i-1]>after_baohan['high'][i]):
                        temp_high = after_baohan['high'][i]
                        temp_num = i-1
                        temp_type = 1
                        i -= 4
                    else:
                        temp_low = after_baohan['low'][i]
                        temp_num = i
                        temp_type = 2
                        i -= 4
            else:
                i -= 1
    return fenxing_type, fenxing_time, fenxing_plot, fenxing_data

def recognition_baohan(Input_k, zoushi, after_baohan):
    ''' 
    判断两根k线的包含关系
    temp_data 包含处理后最后一个k线的数据
    zoushi 包含关系处理中关于走势的记录 是一个dict
    Input_k 是temp_data与新n分钟k线的合集（其实只有两个数据，历史数据最后的数据和最新日期的数据）
    zoushi： 3-持平 4-向下 5-向上
    after_baohan 处理之后的k线
    '''
    import pandas as pd

    temp_data = {}
    temp_data['high'] = Input_k['high'][0]# 实际是k_initialization得出的temp_data的第一个数据
    temp_data['low'] = Input_k['low'][0]

    case1_1 = temp_data['high'] > Input_k['high'][1] and temp_data['low'] < Input_k['low'][1]# 第1根包含第2根（开始的时候i-1时刻k线包含i时刻k线）
    case1_2 = temp_data['high'] > Input_k['high'][1] and temp_data['low'] == Input_k['low'][1]# 第1根包含第2根
    case1_3 = temp_data['high'] == Input_k['high'][1] and temp_data['low'] < Input_k['low'][1]# 第1根包含第2根
    case2_1 = temp_data['high'] < Input_k['high'][1] and temp_data['low'] > Input_k['low'][1] # 第2根包含第1根
    case2_2 = temp_data['high'] < Input_k['high'][1] and temp_data['low'] == Input_k['low'][1] # 第2根包含第1根（开始的时候i时刻k线包含i-1时刻k线）
    case2_3 = temp_data['high'] == Input_k['high'][1] and temp_data['low'] > Input_k['low'][1] # 第2根包含第1根
    case3 = temp_data['high'] == Input_k['high'][1] and temp_data['low'] == Input_k['low'][1] # 第1根等于第2根
    case4 = temp_data['high'] > Input_k['high'][1] and temp_data['low'] > Input_k['low'][1] # 向下趋势
    case5 = temp_data['high'] < Input_k['high'][1] and temp_data['low'] < Input_k['low'][1] # 向上趋势
    if case1_1 or case1_2 or case1_3: 
        if zoushi[-1] == 4:#向下趋势
            temp_data['high'] = Input_k['high'][1]
        else:
            temp_data['low'] = Input_k['low'][1]
            
    elif case2_1 or case2_2 or case2_3: 
        temp_temp = {}
        temp_temp['high'] = temp_data['high']
        temp_temp['low'] = temp_data['low']
        temp_data['high'] = Input_k['high'][1]
        temp_data['low'] = Input_k['low'][1]
        if zoushi[-1] == 4:#向下趋势
            temp_data['high'] = temp_temp['high']
        else:
            temp_data['low'] = temp_temp['low']
            
    elif case3:
        zoushi.append(3)#持平
        pass
    
    elif case4:#向下趋势
        zoushi.append(4)
        after_baohan['high'].append(temp_data['high'])
        after_baohan['low'].append(temp_data['low'])
        temp_data['high'] = Input_k['high'][1]
        temp_data['low'] = Input_k['low'][1]
    elif case5:#向上趋势
        zoushi.append(5)
        after_baohan['high'].append(temp_data['high'])
        after_baohan['low'].append(temp_data['low'])
        temp_data['high'] = Input_k['high'][1]
        temp_data['low'] = Input_k['low'][1]

    return temp_data, zoushi, after_baohan

def k_initialization(security,num = 10*48,n=5):
    '''
    读入回测日期之前的多日k线用以判断之前的趋势
    返回值：
        temp_data 包含处理后最后一个k线的数据
        zoushi 包含关系处理中关于走势的记录
        after_baohan 合并后的k线
        原文内容：
        在向上时，把两K线的最高点当高点，而两K线低点中的较高者当成低点，这样就把两K线合并成一新的K线；
        反之，当向下时，把两K线的最低点当低点，而两K线高点中的较低者当成高点，这样就把两K线合并成一新的K线。
        经过这样的处理，所有K线图都可以处理成没有包含关系的图形。
    '''
    import pandas as pd
    
    x = '%sm'%n
    temp_data = {}
    # zoushi = {}
    after_baohan = {}
    t = {}
    stock=security
    k_data = attribute_history(stock, num, str(x),['high', 'low'], df=False) #读入回测日期之前的历史数据，0位置放时间最早的k线数据
        
    ## 判断包含关系
    after_baohan = {'high':[],'low':[]} #创建一个dict 合并后的k线的最高价、最低价
    t['high'] = k_data['high'][0]
    t['low'] = k_data['low'][0]

    temp_data = t #temp_data初始值是k_data,即0位置的k线的最高价、最低价
    zoushi = [3] # 3-持平 4-向下 5-向上
    for i in xrange(num): #i取从0到num-1，xrange 用法与 range 完全相同，所不同的是生成的不是一个list对象，而是一个生成器generator,效率比较高
        case1_1 = temp_data['high'] > k_data['high'][i] and temp_data['low'] < k_data['low'][i]# 第1根包含第2根（开始的时候i-1时刻k线包含i时刻k线）
        case1_2 = temp_data['high'] > k_data['high'][i] and temp_data['low'] == k_data['low'][i]# 第1根包含第2根
        case1_3 = temp_data['high'] == k_data['high'][i] and temp_data['low'] < k_data['low'][i]# 第1根包含第2根
        case2_1 = temp_data['high'] < k_data['high'][i] and temp_data['low'] > k_data['low'][i] # 第2根包含第1根（开始的时候i时刻k线包含i-1时刻k线）
        case2_2 = temp_data['high'] < k_data['high'][i] and temp_data['low'] == k_data['low'][i] # 第2根包含第1根
        case2_3 = temp_data['high'] == k_data['high'][i] and temp_data['low'] > k_data['low'][i] # 第2根包含第1根
        case3 = temp_data['high'] == k_data['high'][i] and temp_data['low'] == k_data['low'][i] # 第1根等于第2根
        case4 = temp_data['high'] > k_data['high'][i] and temp_data['low'] > k_data['low'][i] # 向下趋势
        case5 = temp_data['high'] < k_data['high'][i] and temp_data['low'] < k_data['low'][i] # 向上趋势
        if case1_1 or case1_2 or case1_3:
            if zoushi[-1] == 4:
                temp_data['high'] = k_data['high'][i]
            else:
                temp_data['low'] = k_data['low'][i]

        elif case2_1 or case2_2 or case2_3:
            temp_temp = {} #临时存储上一次判定后的最高价和最低价
            temp_temp['high'] = temp_data['high']
            temp_temp['low'] = temp_data['low']
            temp_data['high'] = k_data['high'][i]
            temp_data['low'] = k_data['low'][i]
            if zoushi[-1] == 4:
                temp_data['high'] = temp_temp['high']
            else:
                temp_data['low'] = temp_temp['low']

        elif case3:
            zoushi.append(3)
            pass

        elif case4:
            zoushi.append(4)
            after_baohan['high'].append(temp_data['high'])
            after_baohan['low'].append(temp_data['low'])
            temp_data['high'] = k_data['high'][i]
            temp_data['low'] = k_data['low'][i]

        elif case5:
            zoushi.append(5)
            after_baohan['high'].append(temp_data['high'])
            after_baohan['low'].append(temp_data['low'])
            temp_data['high'] = k_data['high'][i]
            temp_data['low'] = k_data['low'][i]
    return temp_data, zoushi, after_baohan#得到最后的最高价和最低价、走势的集合、走势每个数据对应的最高价和最低价

def dp_stoploss(kernel=2, n=10, zs=0.03):
    '''
    方法1：当大盘N日均线(默认60日)与昨日收盘价构成“死叉”，则发出True信号
    方法2：当大盘N日内跌幅超过zs，则发出True信号
    '''
    # 止损方法1：根据大盘指数N日均线进行止损
    if kernel == 1:
        t = n+2
        hist = attribute_history('510050.XSHG', t, '1d', 'close', df=False)
        temp1 = sum(hist['close'][1:-1])/float(n)
        temp2 = sum(hist['close'][0:-2])/float(n)
        close1 = hist['close'][-1]#历史数据最后时刻的收盘价
        close2 = hist['close'][-2]
        if (close2 > temp2) and (close1 < temp1):#首次低于10日均线
            return True
        else:
            return False
    # 止损方法2：根据大盘指数跌幅进行止损
    elif kernel == 2:
        hist1 = attribute_history('510050.XSHG', n, '1d', 'close',df=False)
        if ((1-float(hist1['close'][-1]/hist1['close'][0])) >= zs):#10日跌幅大于zs（3%）
            return True
        else:
            return False
