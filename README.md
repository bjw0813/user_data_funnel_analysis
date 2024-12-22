# 이커머스 로그데이터 - 퍼널분석


# 데이터

![image](https://github.com/user-attachments/assets/5c84457e-06f1-4f8a-80b2-8192cee7694c)


- event_time: 이벤트가 발생한 시각
- event_type: 이벤트 종류
    - view: 상품을 조회
    - cart: 상품을 카트에 추가
    - remove_from_cart: 상품을 카트에서 제거
    - purchase: 구매
- product_id: 상품번호
- category_id: 카테고리번호
- category_code: 카테고리명
- brand: 브랜드명
- price: 상품 가격
- user_id: 고객번호
- user_session: 세션

**데이터 출처: kaggle**

# 분석 목표(분석 전 질문)

- DAU(일간 활성 사용자수) 추이는?
    - 어느 요일에 가장 많이 방문하는지
    
- 고객들의 사이트 체류시간 평균은?
    - 조회만 한 유저, 카트에 담은 유저, 구매까지 한 유저별로 체류시간이 어떻게 다른지
    
- 세 단계(조회, 카트, 구매) 중 어느 단계에서 유저 이탈이 가장 많이 발생하나?

# 데이터 전처리


```python
data["event_time"]=pd.to_datetime(data["event_time"])
```

```python
data.isna().sum()
```

![image.png](image%201.png)

결측값이 너무 많아 `category_id`와 `category_code`는 drop하기로 결정

```python
data.drop(["category_code","brand"],axis=1,inplace=True)
```

```python
data["date_ymd"]=data["event_time"].dt.date
```

DAU(일간 활성 사용자 수)를 위해 date_ymd라는 필드를 따로 만들어줬다

![image.png](image%202.png)

# 데이터 분석

## DAU(일간 활성 사용자수) 추이

```python
dau=data.groupby("date_ymd")[["user_id"]].nunique().reset_index().rename({"user_id":"dau"},axis=1)
dau.head(10)
```

![image.png](image%203.png)

```python
plt.figure(figsize=(12,5))
a=sns.lineplot(x="date_ymd",y="dau",data=dau)

plt.title("DAU")
plt.show()
```

![image.png](image%204.png)

```python
import plotly.express as px 
px.line(data_frame=dau,x="date_ymd",y="dau")
#plotly를 사용
#장점: 그래프가 예쁘고(R같은 느낌), 이미지 저장도 용이하다.
```

![image.png](image%205.png)

```python
dau["date_ymd"]=pd.to_datetime(dau["date_ymd"])
dau["dayname"]=dau["date_ymd"].dt.day_name()#dayname이라는 필드를 만들어준다
avg_dau=dau.groupby("dayname")[["dau"]].mean()
avg_dau.reset_index(inplace=True)
order=["Monday","Tuesday","Wednesday","Thursday","Friday","Saturday","Sunday"]
px.bar(data_frame=avg_dau,x="dayname",y="dau",category_orders={"dayname":order})#화요일에 활성이용자수가 가장 높고, 주말이 가장 적다.
```

![image.png](image%206.png)

## 고객들의 사이트 체류시간 평균은?

이 주제로 분석하면서 마주한 문제는 고객의 체류시간을 계산할 때 세션별로 계산할 것인지, 유저별로 계산할 것인지이다.

나는 유저 아이디보다는 세션이 적합하다고 생각했다. 

내가 알고 싶었던 건 체류시간이였기 때문에 고객별로 시간을 계산한다면 동일 고객이 다른 날 또는 다른 시간에 접속한 세션의 시간까지 합쳐지므로 체류시간이라고 정의한것이 의미가 없어진다고 생각했다.

실제로 찾아보니 실무에서도 체류시간의 경우 세션으로 분석한다는 정보를 얻었다.

```python

duration=data.groupby("user_session")[["event_time"]].agg(["max","min"]).reset_index()
duration["duration"].mean() #Timedelta('0 days 00:59:16.683693260')

#피벗테이블 
pivot=pd.pivot_table(data=data,index="user_session",columns="event_type",values="event_time",aggfunc="count").reset_index().fillna(0)

#카트, 구매 고객 리스트
purchase_session= pivot[pivot["purchase"]>0]["user_session"]
purchase_session.tolist()
only_cart_session= pivot[(pivot["cart"]>0) & (pivot["purchase"]== 0)]["user_session"]
only_cart_session.tolist())

duration.columns=["user_session","max","min","duration"]
```

```python
duration.query("user_session not in @only_cart_session and user_session not in @purchase_session")["duration"].mean()
#조회만 한 이용자들의 체류시간
#Timedelta('0 days 00:38:30.953374025')
```

```python
duration[duration["user_session"].isin(only_cart_session.tolist())]["duration"].mean()
#카트까지만 담은 이용자들의 체류시간
#Timedelta('0 days 01:57:48.286015460')
```

```python
duration[duration["user_session"].isin(purchase_session.tolist())]["duration"].mean()
#구매까지 한 이용자들의 체류시간
#Timedelta('0 days 06:42:21.679333566')
```

## 세 단계(조회, 카트, 구매) 중 어느 단계에서 유저 이탈이 가장 많이 발생하나?

```python
funnel=pd.DataFrame(pivot[["view","cart","remove_from_cart","purchase"]].sum()).reset_index()
funnel
funnel=funnel[funnel["event_type"] != "remove_from_cart"]
funnel.reset_index(drop=True,inplace=True)

#비율 필드 추가
funnel['rate']=[1,funnel["log_count"][1]/funnel["log_count"][0],funnel["log_count"][2]/funnel["log_count"][0]]
#퍼널 그래프 그리기
funnel_chart=px.funnel(data_frame=funnel, x="event_type", y="rate")
funnel_chart.update_traces(texttemplate = "%{value:,.2%}")

```

![image.png](image%207.png)

**카트에서 구매단계로 넘어가는 과정에서 가장 큰 이탈이 발생하는 것을 확인할 수 있다!**

# 분석 결과

**DAU(일간 활성 사용자수)**

- 월 초에서 중순까지 DAU가 증가하는 추세를 보이다가, 중순부터는 정체하는 느낌이 있다.
- 화요일에 가장 많이 방문하고, 주말에 사용자수가 줄어든다.

**사이트 체류시간 평균은?**

- 체류시간 평균은 약 1시간
- 조회만 한 유저는 약 40분, 카트까지 담은 유저는 약 2시간, 구매까지 한 유저는 약 6시간 40분을 체류한다.

**퍼널 분석**

- 상품 조회를 한 후 카트를 담는 단계에서 약 47.3%만 남고
- 카트를 담고 구매를 하는 단계에서 약 9%만 남는다.
- 카트를 담고 구매를 하는 단계에서 이탈이 많이 일어나므로, 해당 단계에서 전환율을 높이기 위한 전략이 필요 → ***결제과정이 불편한지, 회원가입이 까다로운지 등을 체크하고 장바구니 쿠폰이나 장바구니 이벤트 등을 통해 구매 전환율을 높일 수 있는 방안을 고려할 필요가 있다!***

# 느낀점

로그데이터를 경험해보며 세션 등의 개념을 공부할 수 있었다. 또한 분석 문제 정의를 할 때 사람마다 다르게 해석을 할 수 있는 여지가 있을 것 같아 팀이라면 분석을 할때 정확한 소통이 반드시 전제되어야 한다고 느꼈다.(특히 체류시간을 분석하면서,,) 

 또한 plotly를 시각화에 처음 이용해봤는데 그래프가 굉장히 R처럼 깔끔하고 예쁘게 나와서 앞으로도 자주 애용할 것 같고, 이번 경험을 통해 퍼널분석을 실습하며, 단순히 이론만 아는 것이 아닌 체득할 수 있던 개인 프로젝트였다.
