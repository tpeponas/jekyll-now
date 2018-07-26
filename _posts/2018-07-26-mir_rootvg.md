---
layout: post
title: Mirror / unmirror Rootvg Aix
---

Il arrive des cas ou un swap de disque sur le rootvg d'un Aix est nécessaire. Par exemple le rootvg sur hdisk1 vers un hdisk0
La manip consiste à mirrorer sur un nouveau disque et demirrorer l'ancien

Ici le rootvg est sur hdisk1 on veux qu'il soir unique sur hdisk0:

{% highlight bash %}
aix:root /> lspv  
hdisk0          none                                None                        
hdisk1          00002f8bd1e92b12                    rootvg          active      
{% endhighlight %}

On étend le VG avec l'hdisk0:

{% highlight bash %}
aix:root /> extendvg rootvg hdisk0
0516-1254 extendvg: Changing the PVID in the ODM. 
aix:root /> lspv
hdisk0          00002f8bd6dea7e0                    rootvg          active      
hdisk1          00002f8bd1e92b12                    rootvg          active      
{% endhighlight %}

On lance le mirroring an tache de fond:
{% highlight bash %}
aix:root /> mirrorvg -S rootvg hdisk0 
0516-1804 chvg: The quorum change takes effect immediately.
0516-1126 mirrorvg: rootvg successfully mirrored, user should perform
        bosboot of system to initialize boot records.  Then, user must modify
        bootlist to include:  hdisk1 hdisk0.
aix:root /> lsvg -l rootvg
rootvg:
LV NAME             TYPE       LPs     PPs     PVs  LV STATE      MOUNT POINT
hd5                 boot       1       2       2    closed/syncd  N/A
hd6                 paging     4       8       2    open/stale    N/A
paging00            paging     28      56      2    open/stale    N/A
hd8                 jfs2log    1       2       2    open/stale    N/A
hd4                 jfs2       3       6       2    open/stale    /
hd2                 jfs2       36      72      2    open/stale    /usr
hd9var              jfs2       6       12      2    open/stale    /var
hd3                 jfs2       4       8       2    open/stale    /tmp
hd1                 jfs2       1       2       2    open/stale    /home
hd10opt             jfs2       6       12      2    open/stale    /opt
loglv00             jfslog     1       2       2    closed/stale  N/A
hd11admin           jfs2       1       2       2    open/stale    /admin
livedump            jfs2       1       2       2    open/stale    /var/adm/ras/livedump
lv_dump             sysdump    16      16      1    open/syncd    N/A  
{% endhighlight %}

L'etat stale indique que le LV correspondant n'est pas encore mirroré.
Une fois tous les LV a l'état sync:

{% highlight bash %}
aix:root /> lsvg -l rootvg
rootvg:
LV NAME             TYPE       LPs     PPs     PVs  LV STATE      MOUNT POINT
hd5                 boot       1       2       2    closed/syncd  N/A
hd6                 paging     4       8       2    open/syncd    N/A
paging00            paging     28      56      2    open/syncd    N/A
hd8                 jfs2log    1       2       2    open/syncd    N/A
hd4                 jfs2       3       6       2    open/syncd    /
hd2                 jfs2       36      72      2    open/syncd    /usr
hd9var              jfs2       6       12      2    open/syncd    /var
hd3                 jfs2       4       8       2    open/syncd    /tmp
hd1                 jfs2       1       2       2    open/syncd    /home
hd10opt             jfs2       6       12      2    open/syncd    /opt
loglv00             jfslog     1       2       2    closed/syncd  N/A
hd11admin           jfs2       1       2       2    open/syncd    /admin
livedump            jfs2       1       2       2    open/syncd    /var/adm/ras/livedump
lv_dump             sysdump    16      16      1    open/syncd    N/A
{% endhighlight %}

Si on compare le mapping des deux disques on observe une différence: 
{% highlight bash %}
aix:root /> lsvg -p rootvg
rootvg:
PV_NAME           PV STATE          TOTAL PPs   FREE PPs    FREE DISTRIBUTION
hdisk1            active            199         68          08..00..00..20..40
hdisk0            active            199         84          08..00..00..36..40
{% endhighlight %}

Ici c'est le LV lv_dump qui n'est présent que sur hdisk1, pour pouvoir supprimer l'hdisk1 il faut le déplacer.
{% highlight bash %}
aix:root /> migratepv -l lv_dump hdisk1 hdisk0 
{% endhighlight %}

il est bien maintenant dus hdisk0:
{% highlight bash %}
aix:root /> lspv -l hdisk0
hdisk0:
LV NAME               LPs     PPs     DISTRIBUTION          MOUNT POINT
hd11admin             1       1       00..00..00..01..00    /admin
livedump              1       1       01..00..00..00..00    /var/adm/ras/livedump
fslv00                2       2       02..00..00..00..00    /syst
fslv01                20      20      20..00..00..00..00    /mano
lv_dump               16      16      00..00..00..16..00    N/A
hd10opt               6       6       06..00..00..00..00    /opt
loglv00               1       1       01..00..00..00..00    N/A
hd1                   1       1       01..00..00..00..00    /home
hd4                   3       3       00..00..03..00..00    /
hd2                   36      36      00..36..00..00..00    /usr
hd9var                6       6       00..00..06..00..00    /var
hd3                   4       4       00..00..01..03..00    /tmp
hd5                   1       1       01..00..00..00..00    N/A
hd6                   4       4       00..04..00..00..00    N/A
paging00              28      28      00..00..28..00..00    N/A
hd8                   1       1       00..00..01..00..00    N/A
{% endhighlight %}

On peux donc casser le mirror et suprimer l'hdisk1 du rootvg 

{% highlight bash %}
aix:root /> unmirrorvg rootvg hdisk1
0516-1246 rmlvcopy: If hd5 is the boot logical volume, please run 'chpv -c <diskname>'
        as root user to clear the boot record and avoid a potential boot
        off an old boot image that may reside on the disk from which this
        logical volume is moved/removed.
0516-1804 chvg: The quorum change takes effect immediately.
0516-1144 unmirrorvg: rootvg successfully unmirrored, user should perform
        bosboot of system to reinitialize boot records.  Then, user must modify
        bootlist to just include:  hdisk0.
        iaxdbdmz01:root /orage/param> reducevg rootvg hdisk1
aix:root /> lspv
hdisk0          00002f8bd6dea7e0                    rootvg          active      
hdisk1          00002f8bd1e92b12                    None                        
{% endhighlight %}

comme indiqué dans le message , on doit refaire la bootlist et le bosboot sur l'hdisk0:

{% highlight bash %}
aix:root /> bosboot -ad /dev/hdisk0 
bosboot: Boot image is 53276 512 byte blocks.
aix:root /> bootlist -m normal hdisk0
aix:root /> bootlist -m normal -o    
hdisk0 blv=hd5 pathid=0
hdisk0 blv=hd5 pathid=1
hdisk0 blv=hd5 pathid=2
hdisk0 blv=hd5 pathid=3
{% endhighlight %}

Tous est ok on supprime l'hdisk1 de la base ODM.

{% highlight bash %}
aix:root /> rmdev -dl hdisk1
hdisk1 deleted
{% endhighlight %}










