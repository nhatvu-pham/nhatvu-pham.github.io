---
layout: post
title: "LaziTracker - Tracker cho người lười"
date: 2025-10-11
tags: [project]
---

## 1. Cài đặt môi trường

### 1.1. Flash Beaglebone black
- Khi mình mới mua về, mình không kết nối được với beagle bone black. Sau một hồi tìm hiểu thì mới nhận ra là do chưa có flash linux.
- Nếu cũng gặp phải tình trạng này, các bạn cần flash theo các bước sau:
    1. Tải image .iso bất kỳ từ trang chủ của [BeagleBone](https://www.beagleboard.org/distros)
    2. Dùng đầu đọc thẻ nhớ, kết nối vào máy tính, rồi chạy lệnh sau để flash
        ```
        sudo dd if=am335x-debian-13-base-v6.16-armhf-2025-09-05-4gb.img of=/dev/sdX bs=1M status=progress conv=fsync
        ```
        Lưu ý: 
        - Thay **am335x-debian-13-base-v6.16-armhf-2025-09-05-4gb.img**thành file image bạn vừa tải về
        - Thay /dev/sdX bằng device của đầu đọc (ví dụ như sda, sdb). Để tìm device đó, dùng lệnh `lsblk` và xem device nào có dung lượng tương đương thẻ nhớ bạn dùng
    3. Tháo thẻ nhớ, nhét vào đầu đọc của Beaglebone, nhấn giữ nút Boot (gần khe cắm thẻ nhớ) rồi nạp nguồn cho BeagleBone, tới khi các LEDs nhấp nháy thì thả nút. Có thể dùng lệnh `lsusb` để kiểm tra đã boot thành công chưa, nếu đã thành công thì sẽ thấy có device `Linux Foundation Multifunction Composite Gadget`

### 1.2. Kết nối BeagleBone với Internet thông qua PC
- Bình thường, beaglebone chỉ tạo kết nối Ethernet với PC qua cổng USB, để kết nối USB Ethernet này với Internet, thực hiện như sau:
    - Trên PC, chạy lệnh `ip a` để tìm network interface với BeagleBone. Ví dụ
        ```
        11: enx182c65031a75: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        111
    - Nếu interface đang down, kích hoạt bằng lệnh
        ```
        sudo ip link set enx182c65031a75 up
        ```
    - Share connection bằng lệnh
        ```
        sudo nmcli connection add type ethernet ifname enx182c65031a75 con-name BeagleShare ipv4.method shared  
        sudo nmcli connection up BeagleShare
        ```
    - Sau khi chạy các lệnh trên, ta đã tạo được một DHCP server trên PC, tiếp theo, cần chạy lệnh sau trên Beaglebone để lấy IP từ DHCP server
        ```
        sudo dhclient -v usb0
        ```
    - Tuy nhiên, do BeagleBone chưa có IP nên ta cần kết nối serial để chạy lệnh trên, ở đây mình dùng lệnh `screen` để kết nối
        ```
        sudo screen /dev/ttyACM0 115200
        ```
    - Dùng `ip a` để xem IP

### 1.3. Share màn hình từ Beaglebone lên PC
- Vì trong project này mình có dev UI, nên cần màn hình để test. Nhưng không có màn hình nên mình muốn sử dụng laptop để hiển thị   
- Ở đây mình dùng VNCViewer:
    - Trên BeagleBone, cài và chạy VNC server bằng lệnh sau
        ```
        sudo apt install tightvncserver
        vncserver :1
        ```
    - Trên PC, mình tải RealVNC, rồi connect vào BeagleBone theo địa chỉ \[IP vừa config:5901\]

## 2. Các bước thực hiện

### 2.1. Kết nối với LCD ILI9341
- LCD ILI9341 sử dụng SPI. Tuy nhiên, từ phiên bản Debian 12 trở đi, BBB không còn hỗ trợ SPI by default. ~~Phải config device tree để enable SPI~~
- Thay vào đó, mình downgrade và flash version Debian 10 để nạp cho BBB.
- Ngoài ra, cần config chức năng các chân SPI:
  ```
  sudo config-pin P9_18 spi
  sudo config-pin P9_21 spi   # MISO
  sudo config-pin P9_22 spi_sclk   # SCLK
  sudo config-pin P9_17 spi_cs   # CS0
  ```
- Version OS này được cài đặt sẵn python3.7, không còn được hỗ trợ bởi thư viện **Adafruit_ILI9431** mainline, do đó mình phải tự build thư viện python từ source code:
   ```
  git clone   https://github.com/adafruit/Adafruit_Python_ILI9341.git
  cd Adafruit_Python_ILI9341
  sudo python3 setup.py install
  ```
- Hiện ảnh lên LCD
  ```
  import Adafruit_ILI9341 as TFT
  import Adafruit_GPIO.SPI as SPI
  from PIL import Image

  # BBB pins (adjust to your wiring)
  DC = "P9_15"
  RST = "P9_12"
  SPI_PORT = 0
  SPI_DEVICE = 0

  # Initialize display
  disp = TFT.ILI9341(DC, rst=RST,     spi=SPI.SpiDev(SPI_PORT, SPI_DEVICE))
  disp.begin()

  # Load image
  image = Image.open("screen.png")  #  Must be RGB
  disp.display(image)
  ```
  
### 2.2. Chạy script config pins và hiển thị ảnh on start
- Sử dụng system.d, thêm service:
  ```
  sudo nano /etc/systemd/system/ili9341.service
  ```
- Thêm command chạy python script cho service
  ```
  [Unit]
  Description=ILI9341 Display Service
  After=multi-user.target

  [Service]
  Type=simple
  User=debian
  WorkingDirectory=/home/debian
  ExecStart=/usr/bin/python3 /home/debian/spi_display.py
  Restart=on-failure

  [Install]
  WantedBy=multi-user.target
  ```
- Reload daemon và enable service
  ```
  sudo systemctl daemon-reload
  sudo systemctl enable ili9341.service
  sudo systemctl start ili9341.service
  ```
- Check trạng thái của service
  ```
  sudo systemctl status ili9341.service
  ```
