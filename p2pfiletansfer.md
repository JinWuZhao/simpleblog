# 有度客户端P2P文件传输开发过程

## 前言

常用的即时通讯类软件几乎都带有传输文件的功能，聊天过程中随手发个文件过去已经成为了大多数人的习惯。  
有度即时通客户端一开始就具备文件传输功能，但是出于消息漫游的设计要求，传输的文件需要先发送到服务器，再通知接收方客户端，继而下载文件。对于几兆至十几兆的小文件来说，使用完全没问题，但是对于几十兆甚至上百兆、千兆的文件来说体验完全是灾难。浪费服务器流量不说，速度特别慢，同时也浪费服务器磁盘空间。  
对于大家习惯使用的qq来说，发送文件往往是飞速的，尤其是局域网环境下，传输几个G的文件都不在话下。有度作为办公用的聊天工具文件传送需求更为迫切，必须实现一个高效的文件传输机制，我们称之为P2P文件传输。

## 初次设计

由于我们没有P2P传输的开发经验，初始设计以简单为主，首先考虑的是基本可用，方案比较保守。首先我们解决的问题暂时仅限于优化局域网环境的的传输效率，也就是处于同一个子网的两个客户端能够高效的传输数据。  
局域网环境下传输文件其实有很多种方案可以选择，简单点的就tcp、udp，复杂些的比如FTP协议、SMBA协议等等，我们选择了最简单的方案tcp协议。因为tcp首先支持可靠传输，不用考虑太多传输过程中的异常情况，而且库支持广泛，很容易把业务做出来。  
技术方案定好了我们便实现了一个简单的基于tcp的P2P文件传输功能。客户端双方会通过服务器交换子网的IP地址，由发送方建立文件服务，接收方访问发送方的文件服务并建立tcp连接，并且是基于TLS的安全加密连接，文件分片传输以减少内存压力，实时上报传输进度显示到界面上。  

经过测试，速度相比经服务器转发有大幅提升，可用的目的已经达到了。  
但是对比qq文件传输，速度仍然相差不少，发送一个1G大小的文件时间差不多要两分钟，而qq只在数秒内完成。我们不想止步于此，必须要达到相同级别的效率才能不损失用户体验。于是我们开始了进一步的优化。  

## 新传输协议

经过调研我们发现，主流的P2P数据传输实现大多基于UDP，由于tcp协议的复杂性，导致tcp的传输速率有一定瓶颈，我们设想基于UDP的简单可靠传输协议也许能够突破这个瓶颈。  
UDP协议比较简单，单个数据包利用率更高，由于没有拥塞控制，传输速率限制较小，容易实现比较激进的传输策略。UDP协议也可以用简陋来形容，缺少可靠传输的机制、无连接的性质导致我们必须自己实现一套可靠的连接，有一定挑战性。  
由于不想重复造轮子，我们去搜寻了一些开源的基于UDP的可靠传输协议。在其中找到了一个名为KCP的项目，是另一个名为KCPTun项目的子项目。KCPTun是用来进行网络加速的代理工具，可以将原本的tcp数据传输转化为udp传输，并且大幅改善丢包率和延迟，提升访问性能。  
于是我们把kcp协议集成进了有度，但是经测试发现，kcp并没有达到预想速度，调整了很多次参数也无济于事。后来找到了kcp创作者的描述，kcp解决的是复杂网络环境下丢包率高、延时高导致的速度慢的问题，而我们内网传输文件的场景，丢包率和延时都很低，根本发挥不出kcp的优势。

## 对tcp的改观

为了尽可能兼容更多的windows系统版本，我们有度windows客户端的可执行文件是32位的，我在测试过程中偶然实验了下编译为64位的版本，发现速度居然有大幅提升。  
于是我分别测试了的tcp协议与kcp协议的实现在32位和64位版本下的传输速率，结果64位版本的速率明显高过32位版本，而且tcp协议的传输速率比kcp协议还要快一倍。这让我对tcp的印象大幅改观，tcp协议已经优化到满足绝大多数场景的使用了。  

## 改造方案

找到了突破口之后，我们索性就编译了一份64位版本防到安装包里，对于64位系统的用户会自动安装64位版本的可执行文件，改造过程真是异常的简单。经过实际测试，网络带宽允许的情况下至少能够达到40MB/s的传输速率，而原来的实现最多只有10MB/s。  

## 后续改善计划

尽管有了很大的突破，但我们并没有因此停下探索的脚步，我们正在计划使用udp开发一个专用的文件传输协议，看能否突破现有tcp协议的传输速度。而对于大公司内部的多子网环境，跨子网直连可能会有问题，我们还要研究下打洞的实现方案。  