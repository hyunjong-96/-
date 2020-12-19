# 1.sequelize

## (1)M:N

Users와 Organizations table이 M:N관계이다.

```javascript
db.Users=require('./users')(sequelize,Sequelize);
db.Organizations = require('./organizations')(...);
db.Affiliations = require('./affiliations')(...);

db.Users.belongsToMany(db.Organizations,{through:'Affiliations',as:'affiliationed'})

db.Organizations.belongsToMany(db.Users,
{through:'Affiliations',as:'affiliationer'})
```

Users와 Organizations를 다대다관계로 선언해준다.

그리고 Users에서 Organizations의 데이터와 join(inner join)을 해주기 위해서는 

```javascript
await Users.findOne({where:{email:email,password:password},
include:
[{model:Organizations, as:affiliationed}]
})
```

```javascript
[result]

"data":{
	...
	"affiliationed":[
	{
		"id":1,
		"name":"SOPT",
		"Affiliations":{
			"UserId":1,
			"OrganizationId":1,
			...
		}
	}
	]
}
```

Users의 data안에 Organizations를 연결시킨 affiliationed(Affiliations테이블의 별명(as))의 데이터가 속성값으로 join되어있으며 affiationed의 본 테이블인 Affiliations의 데이터도 테이블의 이름을 속성값으로 가지며 join되어 있는것을 확인할수 있다.



## (2)sequelize's Getter & Setter

```javascript
const user = await Users.findOne({where:{CourseOd:courseId}})
```

sequelize에서 courseId로 찾은 데이터는 dataValue, _previousDataValues등으로 나온다.

테이블의 각 attribute들은 get,set메소드를 가지고 있는데 위 와 같은 경우에는 각 attribute들을 get메소드를 통해 값을 가져오는것이다.

```javascript
module.exports=function async(sequelize,DataTypes){
    return sequelize.define('Words',{
        eng:{
            type:DataTypes.STRING,
            get:function(){
                return JSON.parse(this.getDataValue('eng'))
            },
            set: function(val){
                return this.setDataValue('eng',JSON.stringify(val))
            }
        },
        kor:{
            type:DataTypes.STRING,
            get:function(){
                return JSON.parse(this.getDataValue('kor'))
            },
            set: function(val){
                return this.setDataValue('kor',JSON.stringify(val))
            }
        }
    },{
        freezeTableName:true,
        timetables:true
    })
```

words테이블에서 eng attribute는 생성될때 set메소드를 통해 들어온 val파라미터를 통해 javascript값이나 object를 JSON문자열로 변환한다.

sql을 통해 각 attribute값들을 빼올때는 get메소드를 통해 값을 가져오는데 getDataValue('eng')를 통해 engattribute의 값(JSON문자열)을 JSON object형태로 변환하여출력해준다.

이때 get을 통해 가져온 값을 가져오기 위해서는 words.dataValues.eng가 아닌 words.eng를 통해 가져와야지 get메소드를 통해 나온 JSON object값을 받아올수있다.

## (3)through table

```javascript
db.Categories = require('./categories')(sequelize,Sequelize)
db.Contents = require('./contents')(sequelize,Sequelize)

db.Contents.belongsToMany(db.Categories,{through:'CategoryList',foreignKey:'ContentId',otherKey:'CategoryId'})
```

Categories 객체와 Contents객체를 생성하고 Contents와 Categories를 N:M관계로 연결해준다.
이때 belongsToMany메소드를 통해 두 객체를 엮어주는데 CategoryList라는 객체(테이블)는 fk로 Contents의 pk값인 ContentId로, Categories의 pk값인 CategoryId로 필드가 형성이 자동으로 된다.

```javascript
const contents = await Contents.findByPk(contentsId)
const categories = await Categories.findByPk(categoryId)

await contents.addCategories(categories)
```

sequelize에는 CategoryList객체가 보이지는 않지만 의미상 생성이되어있고 mysql를 확인해보면 생성이되어있다.

sequelize에서 CategoryList를 객체로써 사용할수 없기떄문에 서로 관계를 맺어줄 contents와 category 객체를 찾은다음

```
await [fk로 선언된 테이블의 객체].add[CamelCase로 작성한 otherKey 테이블 객체] ([otherKey테이블 객체])
```

로 하면 CategoryList에 각 객체가 id로 연결되어있는걸 확인할수 있다.

## (4) join

### 1)1:1

```
db.Courses=require('./courses')...
db.Contents=require('./contents')...

db
```



### 2)1:N

```javascript
db.Courses=require('./courses')...
db.Contents=require('./contents')...
db.Categories=require('./categories')...

db.Courses.hasMany(db.Contents(onDelete:'cascade'))
db.Contents.belongsTo(db.Courses)

db.Contents.hasMany(db.Categories(onDelete:'cascade'))
db.Categories.belongsTo(db.Contents)

```

hasMany메소드를 통해 target테이블(Contents)에 source(Courses)의 id가 적용된다.

belongsTo메소드를 통해  source테이블(Contents)에 target(Courses)의 id가 적용된다.

```javascript
const course =Courses.findOne({where:{id:courseId},include:[{model:Contents,include:[{model:Categories}]}]})
```

해당 course인스턴스와 관계가 있는 contents와 그 contents와 관계있는 category를 함꼐 join해서 불러올수있다.

### 3)M:N

```javascript
db.Contents = require('./contents')...
db.Categories = require('./categories')...

db.Contents.belongsToMany(db.Categoreis,{through:'CategoryList',foreignKey:'ContentId'})
db.Categories.belongsToMany(db.Contents,{through:'CategoryList',foreignKey:'CategoryId'})
```

```javascript
const content = Contents.findOne({where:{id:ContentsId},include:[{model:Categories,attributes:{through:{attributes:[]}}}]})
```

해당 contents인스턴스에서 관계가 있는 category데이터를 include를 통해 가져올수있고 cactegory요소 안에는 자동적으로 CategoryList(through)이 포함되어있다. CategoryList의 데이터를 포함시키지 않으려면 attributes의 옵션에서 through옵션을 통해 attribute를 빈배열로 선언해주면된다.

## (5)special method

https://sequelize.org/master/manual/assocs.html#many-to-many-relationships

- `fooInstance.getBars()`
- `fooInstance.countBars()`
- `fooInstance.hasBar()`
- `fooInstance.hasBars()`
- `fooInstance.setBars()`
- `fooInstance.addBar()`
- `fooInstance.addBars()`
- `fooInstance.removeBar()`
- `fooInstance.removeBars()`
- `fooInstance.createBar()`

```javascript
const foo = await Foo.create({ name: 'the-foo' });
const bar1 = await Bar.create({ name: 'some-bar' });
const bar2 = await Bar.create({ name: 'another-bar' });
console.log(await foo.getBars()); // []
console.log(await foo.countBars()); // 0
console.log(await foo.hasBar(bar1)); // false
await foo.addBars([bar1, bar2]);
console.log(await foo.countBars()); // 2
await foo.addBar(bar1);
console.log(await foo.countBars()); // 2
console.log(await foo.hasBar(bar1)); // true
await foo.removeBar(bar2);
console.log(await foo.countBars()); // 1
await foo.createBar({ name: 'yet-another-bar' });
console.log(await foo.countBars()); // 2
await foo.setBars([]); // Un-associate all previously associated bars
console.log(await foo.countBars()); // 0
```



## (6)인스턴스 type(객체, 배열)

**findAll**은 sequelize에서 리턴해주는 인스턴스객체는 배열type이므로 

```javascript
const users = Users.findAll()
users.map((v,i)=>{
    console.log(v.dataValues.name)
})
```

배열 인덱스를 통해 객체요소에 접근해야한다.

**findOne**, **findByPk**는 리턴해주는 인스턴스객체는 객체type이므로 바로 찾고자하는 객체요소에 접근할수 있다.

## (7) associatios

has~의 관계메소드는 source테이블의 pk가 target테이블의 fk가 된다.

belongs~의 관계메소드는 target테이블의 pk가 source테이블의 fk가 된다

```javascript
db.Contets = require('./contents')...
db.Categories = require('./categories')...

db.Contents.hasMany(db.Categories,{onDelete:'cascade'})
db.Categories.belongsTo(db.Contents)
```

hasMany메소드를 통해 source(Contents)의 pk가 target(Categories)의 fk가 되고 ContentId로 자동 생성된다.



## (8)Eager Loading, Lazy Loading

```javascript
db.Contets = require('./contents')...
db.Categories = require('./categories')...

db.Contents.hasMany(db.Categories,{onDelete:'cascade'})
db.Categories.belongsTo(db.Contents)
```

위와 같이 Contents와 Categories가 1:M관계로 매핑되어있다.

```javascript
const content = await Contents.findOne({where:{name:contentTitle}})
const category = await content.getCategory()

//이떄의 content는 Lazy Loading으로써 실제로 원하는 경우에만 관련 데이터를 가져 오는 기술

const content = await Contents.findOne({where:{id:contentId},include:Categories})

//include옵션을 용해서 데이터베이스에 하나의 쿼리만 사용해서 인스턴스와 관련 데이터를 가져온다.
```

## (9)Op

sequelize의 model query인 Op

https://sequelize.org/master/manual/model-querying-basics.html

and, or, between, eq등 다양한 query연산자를 사용할수 있다.

```javascript
const Report = require('../model')
const Op = Sequelize.Op
const periodReport = Report.findAll({where:{completeDate:{[Op.between]:[start,end]}}})
```

원하는 attribute에 query문을 넣어줄수있다.

# 2.aws (with mobatermx)

1. key가 있는 디렉토리로 이동
2. chmod 400 [pemKeyName].pem
3. ssh -i '[pemKeyName].pem' ubuntu@[EIP]
4. $sudo apt-get install curl
5. $curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
6.  $sudo apt-get install -y nodejs
7.  $sudo apt-get install build-essential
8. node -v
9. npm -v
10. git clone [git url]
11. 프로젝트dir로 이동후 npm install
12. mobaxterm설정 https://gethlemn.tistory.com/12
13. 프로젝트 최상단에서 $sudo npm install -g pm2
14. $ pm2 start ./bin/www
15. $ pm2 list
16. $ pm2 stop {name || id}

# 3.custom new Error

```javascript
if(user==null){
    var customError = new Error('사용자가 존재하지 않습니다.') //error.message
    customError.name = "CanNotFoundUser" //error.name
    throw customError
}
```

내가 원하는 상황에서 error를 만들어서 던져주면 catch(error)부분에서 내가 만든 에러를 잡아서 사용자에게 res해줄수있다.

# 4.javascript

## (1)push, concat

```javascript
const arr1 = new Array()
const arr2 = new Array()
arr1 = ['cat','dog']
arr2 = ['computer']

arr1.push(arr2) //arr1 = ['cat','dog','computer']
const arr3 = arr1.concat(arr2) 
//arr3 = ['cat','dog','computer']
//arr1 = ['cat','dog']
```

push는 source의 배열에 추가되는것(value, array 구분함)

concat은 source 배열이 아닌 새로운 arr가 리턴된다(value, array구분안함)

## (2)object

추가

```javascript
var contents={}
contents.제목 = 'Node js'
contents.url = 'youtube.com'
contents.words = ['cat','dog']

/*
contents={
	"제목" : 'Node js',
	"url" : 'youtube.com',
	"words" : ['cat','dog']
}
*/
```

```javascript
const word = Words.findByPk(id)
word.dataValues.answer = ['cat','dog']

/*
word={
	...
	"answer" : ['cat','dog']
}
*/
```

sequeluze에서 받아온 인스턴스객체에 추가적인 요소를 넣고싶다면 [인스턴스.dataValues.추가key]

# 5. git

## git rep 새로 생성했을떄

1.git init

2.git status

3.git add .

4.git commit -m "~"

5.git remote add origin "git url"

6.git push origin main

## git rep에서 dir받아올때

1.git clone "git url"

```
"status":"200",
"message":"유사 서비스 가져오기 성공"
"data" :[{
	"id":"serviceId(number)",
	"title":"서비스제목(string)",
	"star":"별점(number)",
	"price":"서비스 가격(number)",
	"image":"메인 이미지(string)",
	"review":"리뷰갯수(number)"
},{"..."},{"..."}]

"star":"별점(number)",
	"review":"리뷰갯수(number)",
	"heart":"좋아요 갯수(number)",
	"price":"가격(number)",
	"layer":"서비스 카테고리 순서?(string)",
	"ServiceImgs:"[{
		"id":"serviceId(number)",
		"img":"서비스 디테일 이미지(string)",
	},{"..."},{"..."}]
```

```
['id', 'title', 'image', 'star', 'review', 'price'
```

