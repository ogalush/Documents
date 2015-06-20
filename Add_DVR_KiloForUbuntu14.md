# OpenStack(Kilo)へDVR(分散ルータ)を入れる.
## どういうもの？
 NetworkNodeの単一障害点を極力解消するために、ComputeNodeにも仮想ルータを入れる機能.  
  
* [High Availability using Distributed Virtual Routing (DVR)](http://docs.openstack.org/networking-guide/deploy_scenario2.html)  
* [Neutronの新機能！DVRとL3HA](http://www.school.ctc-g.co.jp/columns/nakai/nakai57.html)  
* [DVR解説by HP](http://www.slideshare.net/ToruMakabe/20-openstack-neutron-deep-dive-dvr)  
* [DVR公式Doc](http://www.slideshare.net/ToruMakabe/20-openstack-neutron-deep-dive-dvr)  
* [DVR Configuration](http://docs.openstack.org/kilo/config-reference/content/networking-options-dvr.html)  

#手順
 テナントネットワークは[OpenvSwitch + VXLANが必須](http://docs.openstack.org/networking-guide/deploy_scenario2.html)の模様.
