# Water-Softener
##Dumb water softener with home assistant & espHome##

The goal of this project is to describe how to wire a D1 Mini NodeMCU (ESP8266-12) to a water softener (CPED BiO 22 liters) and to configure espHome.

| Hardware required  |  |  |
| ------------- | ------------- | ------------- |
| D1 Mini NodeMCU (ESP8266-12)  | [link to Amazon](https://www.amazon.fr/gp/product/B01N9RXGHY/ref=pe_3044141_189395771_pd_te_s_qp_im?_encoding=UTF8&pd_rd_i=B01N9RXGHY&pd_rd_r=AZ70N9HMVFQYPZTPVFX5&pd_rd_w=o2N3j&pd_rd_wg=VCi3Y)  | ![](https://github.com/tom34/Water-Softener/blob/33341fb78fcdb5e3516713293c75eb1e442d207a/pics-small/NodeMCU%20-%20D1%20Mini-XS.png)|
| 4 Octocoupled relays board  | [link to Amazon](https://www.amazon.fr/gp/product/B078Q8S9S9/ref=ppx_yo_dt_b_search_asin_title?ie=UTF8&psc=1) | ![](https://github.com/tom34/Water-Softener/blob/c4f95d90308fbb6db4f89fb76a1948137767a7ac/pics-small/4%20relays%20module-XS.png)|

<br/>
**How softener work ? <br/> Basic description <br/>
  A softener use 3 valves to operate, each valve is dedicated to a specfic function:
    Valve 1: decompression
    Valve 2: moving train 
    Valve 3: brine suction
                                                                                ,                       ┏━━━━━━━━━━━━━┓       ━Open
                                                                                                        ┃             ┃
 ━━━┫EV1 Decompression┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛     1min    ┗━━━━━  ━Closed (EV3 sur desc.)
                                                                                                        ╎
                                                                                                        ╎ 
                          ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓                     ━Open
                          ┃                                                                             ┃
 ━━━┫EV2-Moving Train ┣━━━┛                               Regeneration                                  ┗━━━━━━━━━━━━━━━━━━━━ ━Closed (EV1 sur desc.)
                          ╎                                                                             ╎
                          ╎                                                                             ╎
                          ┏━━━━━━━━━━━━━┓                                    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━┓                     ━Open
                          ┃             ┃                                    ┃                          ┃
 ━━━┫EV3-Brine suction┣━━━┛    1min     ┗━━━━━━╍┅┅━━━━━━━━━━━━━━━━━━━━━━━━━━━┛           2min           ┗━━━━━━━━━━━━━━━━━━━━ ━Closed (EV2 sur desc.)
                          ╎<  Backwash >╎<  Brine suction + Slow Rince      >╎<       Quick Rince      >╎      
