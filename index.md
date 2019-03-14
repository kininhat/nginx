---
layout: default
---
Bài viết nhằm thảo luận về  **nginx cache cho backend vestacp(nginx + httpd)** dưới góc nhìn của beginner.

Tham khảo [link](https://www.nginx.com/blog/nginx-caching-guide).

Tham khảo [link](http://www.anton-pirker.at/boosting-djangos-performance-with-nginx-reverse-proxy-cache).

Tham khảo [link](https://viblo.asia/p/nginx-server-va-location-block-cach-lam-viec-va-phuong-thuc-dieu-huong-request-3Q75wy3DZWb).


### Các vấn đề  cần lưu ý khi set up nginx cache.

Để  đảm bảo 1 website hoạt động tốt (load nhanh và delay ít) có rất nhiều yếu tố, trong đó cache là 1 trong những yêu cầu giải quyết vấn đề  đó. Nhưng trước khi set up cache, chúng ta cần lưu ý.

*   Cơ chế  xử  lý cache của code: dùng cookie hay plugin ,...
image
*   Đối tượng cách: Dynamic content, Static content
*   Location cached: uri nào được cache, uri nào thì không cần cache
*   ....
Vì vậy, cần xác nhận tối thiểu các yêu cầu như trên trước khi tiến hành.

## Cách cài đặt nginx-cache
#### Khởi tạo
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g
                 inactive=60m use_temp_path=off;
Tham số:
*   proxy_cache_path  => đường dẫn file cache
*   levels  => Cấp độ lưu trữ cache
*   keys_zone  => tên cache và dung lượng cho phép
*   max_size  => dung lượng tối
*   proxy_cache_path=off  => đường dẫn đến file cache

#### Cấu hình
 Sau đây là 1 số  option để  thêm vào file config hay dùng

 proxy_cache my_cache;              #> Gọi cache

 proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;     
 #> Cấp lại cache cũ khi reponse lỗi hoặc cache hết hạn   

 proxy_cache_revalidate on;         #> Load phần thay đổi thay gì toàn bộ    
 ** cache control headers hổ trợ định nghĩa 2 loại:
  + If-Modified-Since (hỏi coi có thay đổi ko)
  + Last-Modified (thay đổi rồi thì load lại vào cache)
 => Tránh load toàn bộ mà chỉ load phần thay đổi

 add_header my_cache $upstream_cache_status; #> Trả về  trạng thái của cache trong phần header, có các trạng thái như:
 * MISS         #> Không trả về  cache trong reponse vì không có cache tồn tại
 * BYPASS       #> Có cache nhưng không dùng
 * EXPIRED      #> Cache hết hạn
 * STALE        #> Server không trả lời -> dùng cache cũ -> không 9 xác
 * UPDATING     #> Cache đang được cập nhật, trong thời gian này -> dùng cache cũ -> không 9 xác
 * REVALIDATED  #> Xác nhận cache reponse -> load phần thay đổi trong cache
 * HIT          #> Trả về  cache trong reponse

 proxy_cache_background_update on;  #> Cập nhật cache trong time update nếu được request, sẽ trả về  cache cũ dùng phối hợp với proxy_cache_use_stale

 proxy_cache_valid 200 30m;         #> Cache những request status 200 với thời gian 30 phút

 proxy_cache_key "$scheme$host$request_uri";  #> Biến được cache (TH này đang dùng mặc định của nginx, có thể  custom thêm tùy trường hợp thực tế)

 proxy_cache_min_uses Times;       #> Request Times lần mới được vào cache

 proxy_cache_bypass $http_cache_control; #> Bỏ qua cache

 proxy_ignore_headers Cache-Control;  #> Bỏ qua cache
 proxy_ignore_headers Set-Cookie;     #> Bỏ qua cache
 proxy_hide_header Set-Cookie;        #> Ẩn header cookie người dùng trong quá trình truy cập

#### Điểm đặt cache và các config
Phần khởi tạo thường được đặt trên block
Thường đặt trong 1 block location
```
location optional_modifier location_match {
    . . .
}
```
- optional_modifier
 - (none) #> Nếu không khai báo gì thì NGINX sẽ hiểu là tất cả các request có URI bắt đầu bằng phần location_match sẽ được chuyển cho location block này xử lí.
 - = #> Khai báo này chỉ ra rằng URI phải có chính xác giống như location_match (giống như so sánh string bình thường).
 - ~ #> Sử dụng regular expression cho các URI.
 - ~* #> Sử dụng regular expression cho các URI cho phép pass cả chữ hoa và chữ thường

Ví dụ
```
location /site {
    . . .
}
```
các request có URI có dạng như sau: /site, /site/page/1, site/index.html sẽ được xử lí thông qua location này.

```
location = /site {
    . . .
}
```
Với 3 URI như bên trên thì chỉ có /site sẽ có thể được xử lí trong directive này, còn /site/page/1 hay /site/index.html thì không.

```
location ~ \.(jpe?g|png|gif|ico)$ {
    . . .
}
```
Các request có đuôi .jpg,.jpeg, .png, .gif, .ico có thể pass qua location này nhưng .PNG thì không

```
location ~* \.(jpe?g|png|gif|ico)$ {
    . . .
}
```
Giống như trên nhưng đuôi .PNG cũng có thể pass.

#### Kiểm tra cache
* Ở trình duyệt có thể  F12
![Image](/assets/img/cached-test.png)

* Ở console
![Image](/assets/img/cached-test2.png)

#### Tham khảo
mkdir /path/to/cache
chown -R nginx:nginx /path/to/cache

proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g
                 inactive=60m use_temp_path=off;
server {
  location / {
      #proxy_cache_bypass $no_cache;
      #proxy_no_cache $no_cache;
      #proxy_cache_bypass $http_cache_control
      proxy_ignore_headers Set-Cookie;
      proxy_hide_header Set-Cookie;
      add_header my_cache $upstream_cache_status;
      proxy_cache my_cache;
      proxy_cache_background_update on;
      proxy_cache_revalidate on;
      proxy_cache_key "$scheme$host$request_uri";
      proxy_cache_valid 200 30m;
      proxy_cache_use_stale error timeout invalid_header updating http_500 http_502 http_503 http_504;
      ....
  }
}
