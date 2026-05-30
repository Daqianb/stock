# stock
import streamlit as st
import pandas as pd
import akshare as ak
from datetime import datetime, timedelta
import plotly.graph_objects as go

# 页面配置
st.set_page_config(page_title="A股资金统计", page_icon="📈", layout="wide")
st.title("📈 A股资金净流入统计系统")

# 初始化会话状态存储数据
if 'data' not in st.session_state:
    st.session_state.data = {}

# 采集今日数据
def fetch_today_data():
    today = datetime.now().strftime('%Y-%m-%d')
    
    with st.spinner("正在采集今日资金流向数据..."):
        # 采集个股数据
        df_stock = ak.stock_fund_flow_individual(symbol="即时")
        df_stock = df_stock[df_stock['净流入-净额'] > 0].sort_values(by='净流入-净额', ascending=False).head(150)
        
        # 采集板块数据
        df_sector = ak.stock_fund_flow_sector(symbol="即时")
        
        # 保存到会话状态
        st.session_state.data[today] = {
            'stocks': df_stock,
            'sectors': df_sector
        }
        
        st.success(f"✅ 数据采集成功！共获取{len(df_stock)}只股票和{len(df_sector)}个板块数据")

# 侧边栏
with st.sidebar:
    st.header("操作")
    if st.button("🔄 采集今日数据", type="primary", use_container_width=True):
        fetch_today_data()
    
    st.divider()
    page = st.radio("选择页面", [
        "每日资金排名",
        "30日出现次数",
        "板块资金排名"
    ])

# 检查是否有数据
if not st.session_state.data:
    st.info("👆 请先点击左侧的'采集今日数据'按钮获取数据")
    st.stop()

# 获取所有可用日期
dates = sorted(st.session_state.data.keys(), reverse=True)
today = dates[0]

# 页面1：每日资金排名
if page == "每日资金排名":
    st.header("每日资金净流入前150名")
    
    selected_date = st.selectbox("选择日期", dates)
    df = st.session_state.data[selected_date]['stocks'].copy()
    
    # 转换单位为亿元
    df['净流入(亿元)'] = (df['净流入-净额'] / 10000).round(2)
    
    # 显示统计
    col1, col2, col3 = st.columns(3)
    col1.metric("统计日期", selected_date)
    col2.metric("上榜股票数", len(df))
    col3.metric("总净流入", f"{df['净流入(亿元)'].sum().round(2)}亿元")
    
    # 显示表格
    st.dataframe(
        df[['代码', '名称', '净流入(亿元)']].reset_index(drop=True),
        column_config={
            'index': '排名',
            '代码': '股票代码',
            '名称': '股票名称'
        },
        hide_index=False,
        use_container_width=True
    )

# 页面2：30日出现次数
elif page == "30日出现次数":
    st.header("近30日出现在前150名次数排名")
    
    # 统计出现次数
    count_dict = {}
    for date in dates[:30]:  # 取最近30天
        df = st.session_state.data[date]['stocks']
        for _, row in df.iterrows():
            key = (row['代码'], row['名称'])
            count_dict[key] = count_dict.get(key, 0) + 1
    
    # 转换为DataFrame
    df_count = pd.DataFrame(
        [(k[0], k[1], v) for k, v in count_dict.items()],
        columns=['股票代码', '股票名称', '出现次数']
    ).sort_values(by='出现次数', ascending=False).reset_index(drop=True)
    
    # 显示统计
    col1, col2, col3 = st.columns(3)
    col1.metric("统计周期", f"近{len(dates[:30])}个交易日")
    col2.metric("上榜股票总数", len(df_count))
    col3.metric("最高出现次数", df_count['出现次数'].iloc[0] if not df_count.empty else 0)
    
    # 显示表格
    st.dataframe(
        df_count,
        hide_index=False,
        use_container_width=True
    )

# 页面3：板块资金排名
elif page == "板块资金排名":
    st.header("每日板块资金净流入排名")
    
    selected_date = st.selectbox("选择日期", dates)
    df = st.session_state.data[selected_date]['sectors'].copy()
    
    # 转换单位为亿元
    df['净流入(亿元)'] = (df['净流入-净额'] / 10000).round(2)
    
    # 显示统计
    col1, col2, col3 = st.columns(3)
    col1.metric("统计日期", selected_date)
    col2.metric("统计板块数", len(df))
    col3.metric("板块总净流入", f"{df[df['净流入(亿元)']>0]['净流入(亿元)'].sum().round(2)}亿元")
    
    # 显示表格
    st.dataframe(
        df[['行业名称', '净流入(亿元)', '领涨股']].reset_index(drop=True),
        column_config={'index': '排名'},
        hide_index=False,
        use_container_width=True
    )
