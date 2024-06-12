# evpn_demo

```
Topology:
                                                                          
            Peering Colo                         Provider Network         
                                     :                                    
                                     :                                    
   ┌───────┐            ┌───────┐    :     ┌───────┐                      
   │       │1   *      1│       │3   :    1│       │                      
   │ Peer1 ├───*┼*──────┤  PE1  ├────:─────┤  PE3  │3                     
   │       │   * *      │       │    :     │       ├─┐                    
   └──────┬┘   * *      └┬────┬─┘    :     └┬──────┘ │                    
         2│    * *      2│    │4     :     2│        │                    
          └────*┼*───────┼─┐  └──────:──────┼─┐      │   *   1┌───────┐   
                *        │ │         :      │ │      └──*┼*───┤       │   
                         │ │         :      │ │         * *   │  P1   │   
                *        │ │         :      │ │      ┌──*┼*───┤       │   
          ┌────*┼*───────┘ │   ┌─────:──────┘ │      │   *   2└───────┘   
         1│    * *        1│   │3    :       1│      │                    
   ┌──────┴┐   * *      ┌──┴───┴┐    :     ┌──┴────┐3│                    
   │       │2  * *     2│       │4   :    2│       ├─┘                    
   │ Peer2 ├───*┼*──────┤  PE2  ├────:─────┤  PE4  │                      
   │       │    *       │       │    :     │       │                      
   └───────┘            └───────┘    :     └───────┘                      
                                     :                                    
                                     :                                    
                                     :                                    
```                                                                        

## This demo is a self-contained containerlab topology.

To run this demo, you only need to have a functioning [containerlab](https://containerlab.dev) installation.

After you have installed [containerlab](https://containerlab.dev), clone this repo, cd into the `evpn_demo` directory and run the following command:

> `sudo clab deploy -t topo.ym -c`

This will launch the entire topology and preconfigure all of the nodes except for PE3 which is without config.

You _could_ copy and paste the contents of the [pe3 config](configs/pe3.cfg) into the PE3 router, or you can follow along with building the evpn each step of the way by following along with [DEMO.md](DEMO.md)
