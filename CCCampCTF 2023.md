
Website: https://play.camp.allesctf.net/
# Web
## 1. Web - Cybercrime Society Club Germany
### Chúng ta biết gì?
- Dockerfile => flag được đặt trong file /usrc/src/app/flag.txt
- Source code
1. Cấu trúc source code
```
├── app
│   ├── static
│   │   ├── app.js
│   │   ├── hacc.jpeg
│   ├── templates
│   ├── app.py
│   ├── flag.txt
│   ├── userdb.json
│   ├── Userdb.py
└── Dockerfile
```
2. Các file quan trọng
<details>
  <summary>File app.py</summary>
  
```
from flask import Flask, request, make_response, render_template, redirect, url_for
import json                                                                                            
from uuid import uuid4 as uuid                                                                         
import random                                                                                          
import string                                                                                          
import hashlib                                                                                         
import time                                                                                            
from Userdb import UserDB                                                                              
import subprocess                                                                                      
                                                                                                       
# Look Ma, I wrote my own web app!

app = Flask(__name__,
            static_url_path='/static',
            static_folder='static',)

userdb = UserDB("userdb.json")
userdb.add_user("admin@cscg.de", 9_001_0001, str(uuid())) 
# print(userdb.get_db())

## Helper Functions 

def check_activation_code(activation_code):
    # no bruteforce
    if "{:0>4}".format(random.randint(0, 10000)) in activation_code:
        return True
    else:
        return False


def delete_accs(emails):
    for email in emails:
        userdb.delete_user(email)
    return True


def error_msg(msg):
    resp = {
        "return": "Error",
        "message": msg
    }
    return json.dumps(resp)


def success_msg(msg):
    resp = {
        "return": "Success",
        "message": msg
    }
    return json.dumps(resp)
    

def get_user(request):
    auth = request.cookies.get("auth")
    if auth is not None:
        return userdb.authenticate_user(auth)
    return None


def validate_command(string):
    return len(string) == 4 and string.index("date") == 0


## API Functions

def api_create_account(data, user):
    dt = data["data"]
    email = dt["email"]
    password = dt["password"]
    groupid = dt["groupid"]
    userid=dt["userid"]
    activation = dt["activation"]

    if email == "admin@cscg.de":
        return error_msg("cant create admin")

    assert(len(groupid) == 3)
    assert(len(userid) == 4)

    userid = json.loads("1" + groupid + userid)
    # print(dt)
    # print(userid)

    if not check_activation_code(activation):
        return error_msg("Activation Code Wrong")
    # print("activation passed")


    if userdb.add_user(email, userid, password):
        # print("user created")
        return success_msg("User Created")
    else:
        return error_msg("User creation failed")


def api_delete_account(data, user):
    if user is None:
        return error_msg("not logged in")

    if data["data"]["email"] != user["email"]:
        return error_msg("Hey thats not your email!")

    # print(list(data["data"].values()))
    if delete_accs(data["data"].values()):
        return success_msg("deleted account")


def api_edit_account(data, user):
    if user is None:
        return error_msg("not logged in")
    
    new = data["data"]["email"]

    if userdb.change_user_mail(user["email"], new):
        return success_msg("Success")
    else:
        return error_msg("Fail")


def api_login(data, user):
    if user is not None:
        return error_msg("already logged in")

    c = userdb.login_user(data["data"]["email"], data["data"]["password"])
    if c is None:
        return error_msg("Wrong User or Password")
    resp = make_response(success_msg("logged in"))
    resp.set_cookie("auth", c)
    return resp


def api_logout(data, user):
    if user is None:
        return error_msg("Already Logged Out")

    userdb[user["email"]]["cookie"] = ""
    userdb.save_db()

    return True
    return render_template('logout.html', title="Logout", user=user)

def api_error(data, user):
    return error_msg("General Error")

def api_admin(data, user):
    if user is None:
        return error_msg("Not logged in")
    is_admin = userdb.is_admin(user["email"])
    if not is_admin:
        return error_msg("User is not Admin")

    cmd = data["data"]["cmd"]
    # currently only "date" is supported
    if validate_command(cmd):
        out = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
        return success_msg(out.stdout.decode())


    return error_msg("invalid command")


actions = {
    "delete_account": api_delete_account,
    "create_account": api_create_account,
    "edit_account": api_edit_account,
    "login": api_login,
    "logout": api_logout,
    "error": api_error,
    "admin": api_admin,
}

## Routes

@app.route("/json_api", methods=["GET", "POST"])
def json_api():
    user = get_user(request)
    if request.method == "POST":
        data = json.loads(request.get_data().decode())
        # print(data)
        action = data.get("action")
        if action is None:
            return "missing action"

        return actions.get(action, api_error)(data, user)

    else:
        return json.dumps(user)

@app.route("/")
def home():
    user = get_user(request)
    if user is not None:
        return redirect(url_for("user"))
    return render_template('home.html', title="Home", user=None)

@app.route("/login")
def login():
    user = get_user(request)
    if user is not None:
        return redirect(url_for("user"))
    return render_template('login.html', title="Login", user=None)

@app.route("/signup")
def signup():
    user = get_user(request)
    if user is not None:
        return redirect(url_for("user"))
    return render_template('signup.html', title="Signup", user=None)

@app.route("/admin")
def admin():
    user = get_user(request)
    if user is None:
        return redirect(url_for("login"))
    # print(user)
    is_admin = userdb.is_admin(user["email"])
    if not is_admin:
        return redirect(url_for("user"))
    return render_template('admin.html', title="Admin", user=user)

@app.route("/user")
def user():
    user = get_user(request)
    if user is None:
        return redirect(url_for("login"))
    # print(user)
    is_admin = userdb.is_admin(user["email"])
    return render_template('user.html', title="User Page", user=user, is_admin=is_admin)

@app.route("/usersettings")
def settings():
    user = get_user(request)
    if user is None:
        return redirect(url_for("login"))
    return render_template('usersettings.html', title="User Settings", user=user)


app.run(host='0.0.0.0', port=1024, debug=False)
```
</details>

<details>
  <summary>File Userdb.py</summary>

  ```
  import json
from uuid import uuid4 as uuid
import random
import string
import hashlib
import time

class UserDB:
    def __init__(self, filename):
        self.db_file = filename

        with open(self.db_file, "r+") as f:
            if len(f.read()) == 0:
                f.write("{}")

        self.db = self.get_data()

    def get_data(self):
        with open(self.db_file, "r") as f:
            return json.loads(f.read())

    def get_db(self):
        return self.db

    def save_db(self):
        with open(self.db_file, "w") as f:
            f.write(json.dumps(self.db))
        self.db = self.get_data()

    def login_user(self, email, password):
        user = self.db.get(email)
        if user is None:
            return None
        
        if user["password"] == hashlib.sha256(password.encode()).hexdigest():
            self.db[email]["cookie"] = str(uuid())
            self.save_db()
            return self.db[email]["cookie"]

        return None


    def authenticate_user(self, cookie):
        for k in self.db:
            dbcookie = self.db[k]["cookie"]
            if dbcookie != "" and cookie == dbcookie:
                return self.db[k]

        return None

    def is_admin(self, email):
        user = self.db.get(email)
        if user is None:
            return False

        #TODO check userid type etc
        return user["email"] == "admin@cscg.de" and user["userid"] > 90000000


    def add_user(self, email, userid, password):
        user = {
            "email": email,
            "userid": userid,
            "password": hashlib.sha256(password.encode()).hexdigest(),
            "cookie": ""
        }

        if self.db.get(email) is None:
            for k in self.db:
                if self.db[k]["userid"] == userid:
                    # print("user id already exists. skipping")
                    return False
        else:
            return False

        self.db[email] = user
        self.save_db()

        return True

    def delete_user(self, email):
        if self.db.get(email) is None:
            print("user doesnt exist")
            return False
        del self.db[email]
        self.save_db()
        return True

    def change_user_mail(self, old, new):
        user = self.db.get(old)
        if user is None:
            return False
        if self.db.get(new) is not None:
            print("account exists")
            return False

        user["email"] = new
        del self.db[old]
        self.db[new] = user
        self.save_db()
        return True
  ```
</details>

### 2. Phân tích source code
- Website có sẵn một tài khoản admin với email là `admin@cscg.de`, có các chức năng: login (với email và password), register (với email, password, groupid, userid, activation_code), edit email, delete account, admin (chỉ admin được phép sử dụng). Với api admin, ta chỉ được phép thực thi câu lệnh `date`. Như vậy, mục tiêu trước mắt là phải chiếm quyền tài khoản admin.

#### API create_account
Chức năng tạo user sẽ kiểm tra:
- email có trùng email admin không
- len(groupid) == 3 và len(userid) == 4
- kiểm tra activation code

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/e5ef99c1-a648-43a6-b05a-db854a71f87a)

Quá trình kiểm tra activation code: trả về true nếu trường "activation" chứa một chuỗi số có 4 ký tự từ 0 đến 10000 (tạo ra ngẫu nhiên) 

=> có thể bypass dễ dàng bằng cách đặt giá trị "0000-0001-...-9999"

```
{
  "action":"create_account",
  "data":{
    "email":"wiener",
    "password":"peter",
    "groupid":"e99",
    "userid":"9999",
    "activation":"0000-0001-0002-0003-0004-0005-0006-0007-0008-0009-...-9998-9999"
  }
}
```

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/7de9e41d-05e4-4967-bf85-0f01c8825b2c)

#### API edit_account
Hàm `api_edit_account` kiểm tra email mới có tồn tại hay chưa, nếu đã tồn tại thì sẽ không được phép thay đổi. 

#### API delete_account
Hàm `api_delete_user` thực hiện:
- So sánh trường "email" có trùng với email của user hiện tại không
- Sau đó, đưa tất cả tham số trong "data" vào hàm `delete_accs`

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/4c2a00f6-a8e3-4990-9ba8-fb5e5789b614)

Như vậy, ta có thể bypass và xóa đi nhiều email. Payload xóa email admin như sau:
```
{
  "action":"delete_account",
  "data":{
    "email":"wiener",
    "email2":"admin@cscg.de"
  }
}
```

#### API admin
Hàm `api_admin` validate command truyền vào và thực thi nó sử dụng `subprocess.run`.
![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/0cfa128c-7028-4dbc-bf1a-f0565af55746)

Hàm `validate_command` kiểm tra độ dài command = 4 và "date" ở index 0
![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/e68ed819-77aa-43de-aa0d-7c8d56bc3a7b)

Với lệnh `date`, ta có thể đọc file bằng lệnh `date -f <file_path>`.

Ta có thể thêm các tham số vào lệnh date bằng cách sử dụng mảng:
```
{
  "action":"admin",
  "data":{
    "cmd":[
      "date",
      "-f",
      "/usr/src/app/flag.txt",
      "-u"
    ]
  }
}
```

#### Hàm is_admin
Hàm is_admin trả về true nếu thỏa mãn cách điều kiện:
- Email là email của admin
- userid > 90000000 (trong api tạo tài khoản, userid sẽ được tạo bằng cách: `userid = json.loads("1" + groupid + userid` => userid)` => sẽ không thể > 90000000 nếu groupid và userid là chữ số thông thường.  

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/ac2cd650-ab59-4b09-bcea-e8bcf7d63787)

Như vậy, ta cần tạo một tài khoản có email=ADMIN_EMAIL và userid > 9000000

### Exploit
Từ những gì đã suy ra, ta có thể tiến hành exploit như sau:
1) Đăng ký một tài khoản với groupid=e99, userid=9999. Khi đó userid sẽ là 1e999999 > 90000000
2) Đăng nhập và xóa tài khoản hiện tại cùng tài khoản admin (bypass như trên)
3) Đăng ký lại và thay đổi email thành email của admin. Khi đó, thỏa mãn 2 điều kiện của hàm `is_admin`
4) Sử dụng API của admin gọi hàm date để đọc file flag.txt, ta được flag đang tìm

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/78c57d56-3a58-49d2-9e86-a0778d2b4693)

# MISC

