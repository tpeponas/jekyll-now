---
layout: post
title: Oracle Memory Usage Aix
---

Pour connaitre l'utilisation mémoire d'une instance oracle + Process utilisateur 

{% highlight bash %}
ps -eo pid,args | grep $ORACLE_SID | awk '{ print $1}' | xargs  -I {} svmon -P {} -O unit=MB,segment=on,filtercat="exclusive shared",filterprop=data,filtertype=working | grep work |  sort -u -k1 | awk ' BEGIN { T=0 } { T=T+$NF} END { print T/1024}'
{% endhighlight %}

Code a optimiser :)
