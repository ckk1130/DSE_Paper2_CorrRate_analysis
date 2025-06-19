# app.py
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="數學答對率分析", layout="wide")

@st.cache_data
def load_data():
    df = pd.read_excel('_DistributionTable_Paper2_答對率分析_T08.xlsx')
    # 補齊空值
    for col in ['Main Topic Name', 'Sub Topic Name', 'Hot Topic Name (Main)']:
        if col not in df.columns:
            df[col] = ""
    return df

df = load_data()

# 頁面導航
st.sidebar.title("數學答對率分析")
page = st.sidebar.radio("功能選擇", [
    "首頁",
    "主題趨勢分析",
    "年度分佈分析",
    "主題熱力圖"
])

if page == "首頁":
    st.title("數學答對率分析平台")
    st.markdown("""
    - **主題趨勢分析**：分析各主題/子題/熱門主題的歷年平均答對率走勢
    - **年度分佈分析**：查看某一年各題的答對率分佈
    - **主題熱力圖**：一覽各主題各年平均答對率
    """)
    st.dataframe(df.head(20))
    st.info("請從左側選單選擇分析功能。")

elif page == "主題趨勢分析":
    st.header("主題趨勢分析")
    topic_type = st.selectbox('分析類型', ['Main Topic', 'Sub Topic', 'Hot Topic'])
    if topic_type == 'Main Topic':
        topic_col = 'Main Topic Name'
    elif topic_type == 'Sub Topic':
        topic_col = 'Sub Topic Name'
    else:
        topic_col = 'Hot Topic Name (Main)'
    
    topic_options = [x for x in df[topic_col].dropna().unique() if str(x).strip() != ""]
    topic = st.selectbox('選擇主題', topic_options)
    
    trend_df = df[df[topic_col] == topic].groupby('Year').agg({
        'Correct rate': ['mean', list],
        'Question No': list
    }).reset_index()
    trend_df.columns = ['Year', 'AvgRate', 'AllRates', 'AllQNo']
    trend_df = trend_df.sort_values('Year')

    fig = px.line(trend_df, x='Year', y='AvgRate', markers=True,
                  title=f'{topic} 各年平均答對率')
    # 自訂滑鼠提示
    fig.update_traces(
        hovertemplate="<b>年份：</b>%{x}<br><b>平均答對率：</b>%{y:.1f}%<br>" +
                      "<b>題號和答對率：</b><br>" +
                      "%{customdata[0]}",
        customdata=[[
            "<br>".join([f"Q{q}：{r}%" for q, r in zip(qnos, rates)])
            for qnos, rates in zip(trend_df['AllQNo'], trend_df['AllRates'])
        ]]
    )
    st.plotly_chart(fig, use_container_width=True)

elif page == "年度分佈分析":
    st.header("年度分佈分析")
    year = st.selectbox('選擇年份', sorted(df['Year'].dropna().unique(), reverse=True))
    year_df = df[df['Year'] == year]
    fig2 = px.bar(year_df, x='Question No', y='Correct rate', 
                  color='Main Topic Name',
                  hover_data=['Main Topic Name', 'Sub Topic Name', 'Hot Topic Name (Main)'],
                  title=f'{year}年各題答對率分佈')
    st.plotly_chart(fig2, use_container_width=True)
    st.dataframe(year_df[['Question No', 'Main Topic Name', 'Sub Topic Name', 'Hot Topic Name (Main)', 'Correct rate']])

elif page == "主題熱力圖":
    st.header("主題熱力圖")
    heat_type = st.selectbox('主題層級', ['Main Topic', 'Sub Topic', 'Hot Topic'])
    if heat_type == 'Main Topic':
        heat_col = 'Main Topic Name'
    elif heat_type == 'Sub Topic':
        heat_col = 'Sub Topic Name'
    else:
        heat_col = 'Hot Topic Name (Main)'
    # 避免空值
    heatmap_df = df[df[heat_col].notnull() & (df[heat_col] != "")]
    if heatmap_df.empty:
        st.warning("此主題層級暫無資料")
    else:
        heatmap = heatmap_df.groupby([heat_col, 'Year'])['Correct rate'].mean().reset_index()
        heatmap_pivot = heatmap.pivot(index=heat_col, columns='Year', values='Correct rate')
        fig3 = px.imshow(
            heatmap_pivot,
            labels=dict(x="年份", y=heat_type, color="平均答對率"),
            color_continuous_scale='RdYlGn',
            aspect='auto',
            title=f'{heat_type} × 年份 答對率熱力圖'
        )
        st.plotly_chart(fig3, use_container_width=True)
        st.dataframe(heatmap_pivot)

