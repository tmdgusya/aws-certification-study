---
title: "SQLAlchemy 조인 종류"
seoTitle: "SQLAlchemy 조인 방법 가이드"
seoDescription: "SQLAlchemy의 다양한 조인 방식과 최적화를 위한 `selectinload`, `joinedload`, `subqueryload` 등 로딩 전략 설명"
datePublished: Tue Sep 30 2025 13:46:44 GMT+0000 (Coordinated Universal Time)
cuid: cmg6m2r3g000202l2a69oabya
slug: sqlalchemy
tags: python, sqlalchemy

---

ORM 을 사용하다보면 어떤 방법으로 연관된 엔티티들을 조회할 방식에 대해 고민을 많이 하게된다. Python 을 사용하면 주로 SQLAlchemy 를 주 ORM 으로 많이 사용하게 되는데, SQLAlchemy 에는 크게 3가지 정도의 연관관계 로딩 방법이 존재한다. 오늘은 그 중 `selectinload` , `joinedload` , `subqueryload` 에 대해서 알아보려고 한다.

## 모델링

일단 테스트를 위한 엔티티 생성은 아래와 같이 조금 복잡하지만 실무에 가까운 모델로 작업해보려고 한다.

* **5개 모델**: User, Post, Comment, Category, Tag
    
* **5개 일대다 관계**: User→Posts, User→Comments, Category→Posts, Post→Comments, Comment→Comments
    
* **3개 다대다 관계**: Post↔Tag, User↔Post(좋아요), User↔User(팔로워)
    
* **총 8개 관계**로 복잡한 조인 테스트 가능
    

## 간단한 조인

만약 우리가 User 가 작성한 Post 를 모두 가져오려면 어떻게 SQLAlchemy 를 통해 쿼리를 작성하면 될까? 일단은 첫번째로 `joinedload` 를 사용해보려고 한다. 공식문서만 보고도 아래와 같이 쉽게 쿼리 작성이 가능하다.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).options(joinedload(User.posts)).where(User.id == 1)
    user = session.execute(stmt).unique().scalars().one_or_none()
    print(user)
```

작성된 DSL 을 보면 `unique` 함수가 붙어 있는걸 알수 있다. 이게 당장은 의문일 수 있지만 아래 쿼리를 보면 해결되니 한번 같이 보도록 하자. 이 경우 작성되는 쿼리는 아래와 같이 작성된다.

```sql
SELECT users.id, users.username, users.email, users.full_name, users.bio, users.is_active, users.created_at, users.updated_at, posts_1.id AS id_1, posts_1.title, posts_1.slug, posts_1.content, posts_1.published, posts_1.view_count, posts_1.created_at AS created_at_1, posts_1.updated_at AS updated_at_1, posts_1.author_id, posts_1.category_id 
FROM users LEFT OUTER JOIN posts AS posts_1 ON users.id = posts_1.author_id 
WHERE users.id = %(id_2)s
```

이렇게 쿼리를 날리게 되면 데이터는 어떻게 나올까?

```markdown
| id | username | email | full\_name | bio | is\_active | created\_at | updated\_at | id\_1 | title | slug | content | published | view\_count | created\_at\_1 | updated\_at\_1 | author\_id | category\_id |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 36 | Someone vote line certainly benefit movement bag trade | pretty-maintain | Citizen power expert finish. Just PM improve oil task camera sell. Particularly course what wonder.<br/>Choice floor simply participant natural identify. Practice base share peace. Development ago cost.<br/>South specific occur teach education put else painting. Guy enjoy inside book TV data usually everybody.<br/>Development authority red another until final. Hundred forward safe.<br/>Candidate play nice. Long chance successful against certain. Approach between sister foot.<br/>Future his be maintain present. Behind he see under memory lay page.<br/>Poor speech far note quite education country spring. Heart oil challenge minute law. Together form stuff agree.<br/>Newspaper experience explain base necessary begin. Tax ok black through some assume yourself.<br/>Sound evening save phone find reflect forget. Idea year father. Because same morning.<br/>News join everybody. Road sense star minute state. Which million Mr white relationship summer.<br/>Side travel senior already save history plan. Even different expect year series hospital wide.<br/>Material oil doctor marriage agent education ability water.<br/>Myself anyone out feel. Drug debate low development woman. Their bed clearly recently team friend face.<br/>Drive machine myself pass.<br/>Close foreign medical city trial wish. General land yard truth.<br/>Not foot meeting to. Treatment decade onto sea once. Democratic coach result between entire service performance.<br/>Key full mean fact.<br/>Speech suffer PM painting blue someone billion. Bed out table bring. Report back pull child member media through. Gas option put.<br/>Participant significant activity number represent appear. National item society ground home.<br/>Exist vote just cut stay everyone area. Can finally would west difference remember experience.<br/>Sing choose door would move including affect. Treat around as culture same mind. Few cover impact police others movement traditional.<br/>Whether everyone much their. Never young eye use describe final physical describe.<br/>Charge about foreign fly surface. | true | 4881 | 2025-07-29 15:20:38.395593 | 2025-09-30 12:15:50.722468 | 1 | 7 |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 49 | Wish cover collection when certain four role | as-late-go-argue | Sure civil trial send. Catch also here beat throughout manager pressure. Computer worry interest direction ask about anything over.<br/>Seek field shake as represent factor total. Buy particular sport reach successful lay mean. Pm trial continue.<br/>Cover turn western senior body billion. Whatever stay go floor.<br/>Story mention high identify white house. Agree democratic be if detail.<br/>Baby page spend physical. Crime ago performance standard brother.<br/>Present market try arrive. Cover newspaper statement world. Kid purpose full four several.<br/>National be challenge especially.<br/>Current third her must boy throughout. Vote loss because side nor try suffer. Size nothing fine room.<br/>Help heavy enter point school. Commercial hospital appear since usually fact suddenly. Next management care important represent share.<br/>Necessary yes case center. Lot employee sell question fear. Without protect likely evidence.<br/>Road media program decade. Within chance plant do able worry easy. Culture dark tax it.<br/>Address may community world pressure receive theory. Production yet ready protect.<br/>Prepare onto coach call. Author in executive senior this. Site no manager them.<br/>Agree skill state edge. Fine learn education expert what our national reflect.<br/>Will single back family country gun others. Box degree heart memory lose space. Mr article news every continue pattern begin.<br/>Wait general hundred doctor management. Old exactly brother would guess develop.<br/>To join morning cup cell stop. Seat instead team writer with law. Nature decade rise issue town.<br/>Enter enter improve rather gun reveal discuss. Body my program later. Center government although trade knowledge.<br/>If close popular she. Card everything floor message stuff admit leg. Fund lawyer south.<br/>Position focus into four we house. Capital none simple play what number they whatever.<br/>Heart office care. Authority magazine great involve. Gas century everything.<br/>Name lose all support. Type business man keep bed southern Congress. | true | 2560 | 2025-06-05 21:02:23.312461 | 2025-09-30 12:15:50.722472 | 1 | 3 |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 97 | Myself continue group use generation | present-phone-point | Social film every fish free. Hundred instead technology political there fire travel child. Teach go safe out throw drive under.<br/>Career public hundred despite about within. Debate more media ball the.<br/>Argue blood field heavy very number friend. Population short past almost. Machine strategy support. Health fund foot return Republican pattern speech three.<br/>Special against that industry throw respond girl. Material able debate near third. Serve including phone wall central some herself.<br/>Oil get risk turn. Employee himself analysis difference.<br/>Wonder whose about agent article sell. Their cost democratic according describe central.<br/>Pay happen several put one far. Simply claim through can product simple within.<br/>Concern appear require pretty election letter. Material responsibility him man thing where. Control quite unit among recently at push bed.<br/>Ask growth anything at fund save her. Suffer worker he kitchen inside billion then. Relationship talk floor.<br/>Property discussion development story age. Summer glass difficult leave sport into letter. Movement fire significant religious glass.<br/>Gun mission central feel lose light military about. After house husband girl admit difficult. Particular notice follow option.<br/>Inside performance around look garden feel necessary. For worry relationship of always job cut.<br/>Drug top source economy. Billion experience civil number.<br/>Ball you my provide cause agreement door begin.<br/>Do only can TV want. Various nation purpose move. Analysis wife economy perhaps tough reveal foreign.<br/>Feel according scene base sometimes soon thousand. Popular health tree something. Real different southern kitchen.<br/>Buy analysis trade very arrive create. Price we short while too. Book gun miss open collection along.<br/>Adult thousand building time write explain. Able company born truth whose.<br/>Bag television keep future whole. Expect cell station magazine decade change. Agreement couple easy idea and politics know say. | true | 2112 | 2025-09-14 11:43:32.026224 | 2025-09-30 12:15:50.722494 | 1 | null |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 100 | Writer network strong figure again happy | pay-try-why-soon | Woman traditional painting clear office already study huge. Former market art data statement seat. Back commercial recognize heart summer course throughout.<br/>Provide election college like quite key stuff.<br/>Pass beat hundred shoulder. Lawyer citizen open state source entire executive always.<br/>Old across suffer trial particularly dinner side. Ability great loss eye very may front soon.<br/>Body matter then play world worry tax expect. Study better coach bank quickly his trip last.<br/>Movie newspaper south west hand shoulder our. Along laugh whatever. Use major many outside. Support design oil computer health.<br/>Both night role list environmental data. Despite newspaper affect interview explain fight two nothing. Professor dog page couple again give live girl.<br/>Trial partner receive improve. Store fly eat art book wonder political. Somebody particular dark.<br/>Now new population. Item student side another. Old what professor success structure mean.<br/>Little make red. Season performance actually entire small.<br/>Plan knowledge clearly father. Upon hour none fill.<br/>Plan around group stock a market customer.<br/>Budget language forward people card much teacher grow. Theory work economic soldier fast Republican customer. Claim involve challenge big tree center office realize.<br/>Figure thing leg test medical bag. Single future happen customer father. Rock different check old that point necessary.<br/>Single watch sign. Before security take mission social entire guy.<br/>Democrat table clearly where take wall black single. Small middle they Mrs may my heavy.<br/>Various go organization claim skin writer poor. Reduce agency current way win.<br/>Almost do small place use end. Government season morning stuff good next describe true.<br/>Treatment another let answer each those then. Pattern prove attorney today example act industry somebody. Interest weight painting on group sure.<br/>Number section fine various quickly.<br/>Discover customer ground their certain Congress. Black any democratic girl. Listen pass involve type. | true | 9993 | 2025-09-17 05:01:05.686101 | 2025-09-30 12:15:50.722495 | 1 | 4 |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 123 | Final lay lead home culture run drug | budget-loss-weight | Serve shoulder likely report. Must whole grow front peace.<br/>Out per quite night among investment else travel. Loss serious heavy green.<br/>Service her hotel company win event. Me clearly that course walk serve eat. Goal blue ground study talk happy.<br/>Attorney glass try throw exist condition military. Build building loss in without.<br/>Personal particular most upon major store. Card store level by owner. Bad describe baby play.<br/>Citizen page hospital pick lose culture. Theory term student so prevent.<br/>Decision food cut type exactly. Institution anyone plant build yet technology.<br/>Music head article enter partner turn heavy whole. Anyone difference decision. Like peace relationship mission participant.<br/>Evidence coach animal something.<br/>Mind this successful bad fund already pull. Foreign hair bill decision.<br/>Have budget of challenge even guess low. House career common share thought most bit. Teacher picture say former.<br/>Weight business find Democrat discover.<br/>Need consider live. Assume bad once pretty. Discuss have race allow.<br/>Artist bed doctor miss week sing order. Green memory agreement staff candidate imagine management.<br/>Activity test rate find room ask. Dinner across which including water miss spend subject. Start decide recent section.<br/>Team two paper mission behavior culture specific. There line north use scientist affect. Spend begin skill yet defense.<br/>Including cultural campaign.<br/>Itself firm from point. High close security body.<br/>Cut interesting control might hotel. Reach claim win realize natural.<br/>Small beautiful increase church. Interesting move him professional employee eat. Summer talk physical rock though.<br/>Land yourself movement almost admit. Crime sure information stand cover me significant.<br/>Evening factor add single couple large result. Take under official itself idea example.<br/>Measure may country quite edge beat. Concern dog dinner us. Term scene similar.<br/>Life reflect himself right the officer investment represent.<br/>Add out however score today. Table which as system. | false | 7750 | 2025-05-12 10:38:35.722598 | 2025-09-30 12:15:50.722502 | 1 | 3 |
| 1 | amyaguilar | alewis@example.com | Andrew James | Entire collection Democrat education first. Senior thousand might than news play. | true | 2025-08-31 07:50:11.956596 | 2025-09-30 12:15:50.381909 | 478 | Fine garden over smile read agent plant | across-fund-board | Area argue position term article. Democratic might trial. Operation might necessary animal certain west.<br/>Music program push wind. Light three traditional add.<br/>Include floor which offer write fill provide away. Mention my act mother long not knowledge. Top nothing very ask. Create garden with case present.<br/>Team necessary citizen situation measure. Seem phone ground if. Do see institution chair build.<br/>Hard bill simple car at throughout. Republican soldier range down get affect special. Everyone ability father coach class president threat. Start end special establish.<br/>However stock news floor agency water. Style top manage with hard question whether down.<br/>At anything station itself data. Street American must let difficult. Who official all style so.<br/>Interview that peace majority cultural staff successful. Sit particularly specific moment occur.<br/>Section growth reveal world. Base glass technology work address task yes hour.<br/>Understand thousand least them gun glass hand. Everyone since into Congress soldier wonder newspaper. Admit down garden speech save.<br/>Try yeah true poor unit discuss. Mission structure tend would option. Public interesting much population.<br/>People issue arm college.<br/>Act art sense century hand Republican. May job order. Degree enjoy describe appear past agent most.<br/>Sometimes inside if quite product close wish.<br/>Different political official prove mention compare where. What produce order approach. Begin recent ahead cost. Health any letter.<br/>Nor specific citizen must describe. Bed than there address physical.<br/>Meet entire cost activity hope wait easy member. Large note new note require son great.<br/>Model test prepare significant husband onto into whatever. End allow laugh out best rate data job. Him instead sort north store.<br/>Improve television church minute young. Something break executive cut together hit law. | true | 3827 | 2025-06-11 18:54:02.573873 | 2025-09-30 12:15:50.722608 | 1 | 1 |
```

데이터를 보면 **동일한 user 데이터(우리가 조회한 user 1) 가** 유저가 작성한 포스트의 개수만큼 반복되는 것을 확인할수 있다. 즉, 기본적으로 **LEFT OUTER JOIN** 으로 동작한다는 것이다. 이렇게 되면 어떤 문제가 있을까? 받아온 쿼리 byte 를 serialize 하다가 동일한 user 의 데이터가 여러번 생기게 될수 있다. 따라서, SQLAlchemy 에서는 이를 `unique` 로 제어한다.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).options(joinedload(User.posts)).limit(1000)
    users = session.execute(stmt).unique().scalars().all()
    all_posts = [post for user in users for post in user.posts]
    print(len(all_posts)) # 500
```

만약 위와 같은 쿼리로 200명의 유저를 Post 와 함께 Join 하게 된다면 얼마만큼의 시간이 걸릴까? 데이터가 500개로 아직 크지 않아서 약 `40.84ms` 가 소요되었다.

성능상으로 크게 문제가 없어보이지만 다르게 생각하면 바뀌지 않는 data 에 대한 byte 를 끊임없이 소모함을 알수 있다. 사실 저 정보에서 유저정보는 유니크하기 때문에 굳이 더 받아올 필요가 없다. 더 좋은 방식이 없을까? 이럴때 SQLAlchemy 에서 권장하는 `selectinload` 를 써볼수 있다.

## selectinload

이건 뭐지? 싶을 수 있지만 말 그대로 `SELECT IN …` 쿼리를 이용해서 로드하는 방식이다. 일단 공식문서에 따라 코드로 작성해보면 아래와 같이 작성해볼수 있다.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).options(selectinload(User.posts)).limit(1000)
    users = session.execute(stmt).scalars().all()
    all_posts = [post for user in users for post in user.posts]
    print(len(all_posts)) # 500
```

이 방법은 처음에 User 를 먼져 로딩해온다. User 를 먼져 로딩해오고 그 이후에 id 로 `select in` **쿼리**를 날려야 하기때문이다.

```sql
SELECT users.id, users.username, users.email, users.full_name, users.bio, users.is_active, users.created_at, users.updated_at 
FROM users 
 LIMIT %(param_1)s
```

이렇게 해서 가져온 id 를 통해서 다시 post 테이블에 쿼리를 날리게 된다.

```sql
SELECT posts.author_id AS posts_author_id, posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.category_id AS posts_category_id
FROM posts
WHERE posts.author_id IN (1, 2, 3, ...)
```

이 방식의 장점은 무엇일까? 바로 위에서 언급했듯이 동일한 user 에 대한 정보를 N 번 반복해서 소비하지 않는 다는 점이다. 만약 Join 을 통해 반복되는 데이터가 크다면 이는 네트워크의 대역폭을 절감해주는 역할 또한 해줄수 있다. 따라서, SQLAlchemy 에서 가장 권장하는 방법이며 필자도 이 방법을 가장 많이 쓴다.

아주 간단하고 좋아보이는 방법이지만 무슨 문제가 있을까? 만약에 User 에 연관된 `post`, `liked_post`, `comments`, `following` 등의 엔티티를 함께 로드해야 된다고 해보자.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).options(
        selectinload(User.posts).selectinload(Post.comments),
        selectinload(User.comments),
        selectinload(User.liked_posts),
        selectinload(User.following)
    ).limit(1000)
    users = session.execute(stmt).scalars().all()
```

이렇게 되면 총 몇개의 쿼리가 나갈까? 아마도 User 한번 각 연관관계들 별로 한번씩 쿼리가 발생하여 총 6번의 쿼리가 나갈 것이다.

```sql
SELECT users.id, users.username, users.email, users.full_name, users.bio, users.is_active, users.created_at, users.updated_at 
FROM users 
 LIMIT 1000

SELECT users_1.id AS users_1_id, posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.author_id AS posts_author_id, posts.category_id AS posts_category_id 
FROM users AS users_1 JOIN post_likes AS post_likes_1 ON users_1.id = post_likes_1.user_id JOIN posts ON posts.id = post_likes_1.post_id 
WHERE users_1.id IN (1, 2, 3, ...)

SELECT users_1.id AS users_1_id, users.id AS users_id, users.username AS users_username, users.email AS users_email, users.full_name AS users_full_name, users.bio AS users_bio, users.is_active AS users_is_active, users.created_at AS users_created_at, users.updated_at AS users_updated_at 
FROM users AS users_1 JOIN followers AS followers_1 ON users_1.id = followers_1.follower_id JOIN users ON users.id = followers_1.followed_id 
WHERE users_1.id IN (1, 2, 3, ...)

SELECT posts.author_id AS posts_author_id, posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.category_id AS posts_category_id 
FROM posts 
WHERE posts.author_id IN (1, 2, 3, ...)

SELECT comments.author_id AS comments_author_id, comments.id AS comments_id, comments.content AS comments_content, comments.is_approved AS comments_is_approved, comments.created_at AS comments_created_at, comments.updated_at AS comments_updated_at, comments.post_id AS comments_post_id, comments.parent_id AS comments_parent_id 
FROM comments 
WHERE comments.author_id IN (1, 2, 3, ...)

sqlalchemy.engine.Engine SELECT comments.post_id AS comments_post_id, comments.id AS comments_id, comments.content AS comments_content, comments.is_approved AS comments_is_approved, comments.created_at AS comments_created_at, comments.updated_at AS comments_updated_at, comments.author_id AS comments_author_id, comments.parent_id AS comments_parent_id 
FROM comments 
WHERE comments.post_id IN (1, 2, 3, ...) # 여기는 post_id 이다.
```

엔티티를 한번에 가져와서 사용하기 때문에 SQLAlchemy 공식문서에 나와있듯이 대부분의 상황에서 효율적이다. `joinedload` 에서 문제가 되었던 중복 데이터의 컨슘또한 해결이 되었다. 하지만 항상 정답인 것은 없으므로 단점을 한번 생각해봐야 한다. 첫번째로 가장 쉽게 보이는 단점은 아래와 같다.

### 네트워크 왕복 횟수 증가

기존 **joinedload** 에서는 모든 데이터를 한번에 가져오므로 하나의 쿼리로 충족이 가능하다. 다만, 위와 같이 연관관계가 복잡해지고, 더 많은 데이터를 가져올수록 더 많은 중복이 발생하므로 이는 데이터의 크기를 급격하게 증가시킬수 있다. 따라서 이 부분에서도 간단하게 `selectinload` 를 사용하는 것이 성능이 좋을 것이다.

### 부모가 10000 개인데 자식은 10개인 경우?

극단적인 상황을 생각해보자 만약 부모 row 는 10000 개인데 자식 N 개에 대한 row 는 자식당 10개 정도라고 해보자. 그렇다면 우리는 자식 10N 개를 조회하기 위해 10000N 개의 PK 를 IN 절에 넣게된다. 즉, 이 경우 상당히 비효율적으로 쿼리가 발생하게 된다.

```sql
SELECT users.id, users.username, users.email, users.full_name, users.bio, users.is_active, users.created_at, users.updated_at 
FROM users 
 LIMIT 1000

SELECT users_1.id AS users_1_id, posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.author_id AS posts_author_id, posts.category_id AS posts_category_id 
FROM users AS users_1 JOIN post_likes AS post_likes_1 ON users_1.id = post_likes_1.user_id JOIN posts ON posts.id = post_likes_1.post_id 
WHERE users_1.id IN (1, 2, 3, ..., 10000)

SELECT users_1.id AS users_1_id, users.id AS users_id, users.username AS users_username, users.email AS users_email, users.full_name AS users_full_name, users.bio AS users_bio, users.is_active AS users_is_active, users.created_at AS users_created_at, users.updated_at AS users_updated_at 
FROM users AS users_1 JOIN followers AS followers_1 ON users_1.id = followers_1.follower_id JOIN users ON users.id = followers_1.followed_id 
WHERE users_1.id IN (1, 2, 3, ..., 10000)

SELECT posts.author_id AS posts_author_id, posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.category_id AS posts_category_id 
FROM posts 
WHERE posts.author_id IN (1, 2, 3, ..., 10000)

SELECT comments.author_id AS comments_author_id, comments.id AS comments_id, comments.content AS comments_content, comments.is_approved AS comments_is_approved, comments.created_at AS comments_created_at, comments.updated_at AS comments_updated_at, comments.post_id AS comments_post_id, comments.parent_id AS comments_parent_id 
FROM comments 
WHERE comments.author_id IN (1, 2, 3, ..., 10000)

sqlalchemy.engine.Engine SELECT comments.post_id AS comments_post_id, comments.id AS comments_id, comments.content AS comments_content, comments.is_approved AS comments_is_approved, comments.created_at AS comments_created_at, comments.updated_at AS comments_updated_at, comments.author_id AS comments_author_id, comments.parent_id AS comments_parent_id 
FROM comments 
WHERE comments.post_id IN (1, 2, 3, ..., 10000)
```

이러한 경우에는 어떠한 loading 전략(option)을 사용하면 좋을까? SQLAlchemy 에는 이러한 상황에 사용하기 위한 `subqueryload` 가 존재한다. 아주 쉽게 저 IN 절의 PK 부분을 subquery 로 대체하는 방식이다.

## subqueryload

```sql
SELECT posts.id AS posts_id, posts.title AS posts_title, posts.slug AS posts_slug, posts.content AS posts_content, posts.published AS posts_published, posts.view_count AS posts_view_count, posts.created_at AS posts_created_at, posts.updated_at AS posts_updated_at, posts.author_id AS posts_author_id, posts.category_id AS posts_category_id, anon_1.users_id AS anon_1_users_id 
FROM (
    SELECT users.id AS users_id 
    FROM users 
    LIMIT 1000
) AS anon_1 JOIN posts ON anon_1.users_id = posts.author_id
```

이렇게 되면 만약 내부의 user 가 10000명일 경우 IN 절에 대한 쿼리에 대한 용량을 줄일 수 있게 된다.

### 만약 자식 collection 에 대한 filter 를 하고 싶은 경우?

컬렉션을 로딩하다보면 이런 요구 사항들이 생기곤 한다. `유저가 작성한 포스트 중에 publish 된거만 보고 싶어` 라는 요구사항들이 생긴다. 이럴경우 `contains_eager` 를 이용하면 쉽게 구현이 가능하다.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).join(User.posts).filter(Post.published == True).options(contains_eager(User.posts))
    users = session.execute(stmt).scalars().unique().all()
    all_posts = [post for user in users for post in user.posts]
    print(len(all_posts)) # 457
```

`contains_eager` 는 join 으로 구현되어 공식문서에 나온대로 구현하면 쉽게 따라할수 있다. 이 방법은 아주 간단하게 앞에서 join 으로 가져온 데이터를 단순히 wrapping 한다고 생각하면 쉽다. 하지만 join 을 해야하므로 `selectinload` 를 쓰는 부분에서 아쉬울 수 있다.

### selectinload 에서 자식 컬렉션 필터링

다행이도, SQLAlchemy 가 업데이트 되면서 `selectinload` 에서도 가능한데 단순히 `and_` 를 사용하면 된다.

```python
@measure_time
def test_get_posts():
    session = get_session()
    stmt = select(User).options(
        selectinload(User.posts.and_(Post.published == True))
    ).limit(1000)
    users = session.execute(stmt).scalars().all()
    all_posts = [post for user in users for post in user.posts]
    print(len(all_posts)) # 457
```

## 마치며

테스트할 데이터나 테이블을 쉽게 seeding 하고 싶으면 아래 github repository 를 클론하면 된다. Vibe coding 으로 만들어졌고, 필자도 테스트할때만 코드를 바꿔가며 이용해서 하나 function 을 만들고 그냥 Query 나 성능 테스트를 진행하기에 딱 좋은 Repo 이다.

[https://github.com/tmdgusya/example-sqlalchemy](https://github.com/tmdgusya/example-sqlalchemy)