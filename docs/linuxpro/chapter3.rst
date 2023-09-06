
Chapter 3 : Memory Management
################################################
메모리  관리는  커널에서  가장  복잡하면서도  동시에  가장  중요한  부분  중  하나입니다.
수행할  작업을  수행하려면  프로세서와  커널이  매우  긴밀하게  협력해야  하기  때문에  프로세서와  커널  간의  협력이
강력히  요구되는  것이  특징입니다.
1장에서는  메모리  관리  구현  시  커널이  사용하는  다양한  기술과  추상화에  대한  간략한  개요를  제공했습니다.
이  장에서는  구현의  기술적  측면을  자세히  검토합니다.
3.1 Overview
==========

메모리  관리  구현은  다양한  영역을  다룹니다.
메모리의  물리적  페이지  관리.
 􀀁  메모리를  큰  덩어리로  할당하는  버디  시스템입니다.
 􀀁  더  작은  메모리  청크를  할당하기  위한  slab,  slub  및  slob  할당자.
 􀀁  비연속적인  메모리  블록을  할당하는  vmalloc  메커니즘입니다.
 􀀁  프로세스의  주소  공간

우리가  알고  있듯이  프로세서의  가상  주소  공간은  일반적으로  Linux  커널에  의해  두  부분으로  나뉩니다.
아래쪽과  큰  부분은  사용자  프로세스에  사용  가능하고  위쪽  부분은  커널용으로  예약되어  있습니다.
하위  부분은  컨텍스트  전환(두  사용자  프로세스  간)  중에  수정되는  반면,
가상  주소  공간의  커널  부분은  항상  동일하게  유지됩니다.
IA-32  시스템에서  주소  공간은  일반적으로  사용자  프로세스와  커널  간에  3:1의  비율로  나뉩니다.
가상  주소  공간이  4GiB인  경우  사용자  공간에는  3GiB,  커널에는  1GiB를  사용할  수  있습니다.
이  비율은  관련  구성  옵션을  수정하여  변경할  수  있습니다.
그러나  이는  매우  특정한  구성  및  애플리케이션에만  이점이  있습니다.
추가  조사를  위해  지금은  비율을  3:1로  가정하지만  나중에  다른  비율로  다시  적용하겠습니다.
사용  가능한  물리적  메모리는  커널의  주소  공간에  매핑됩니다.
따라서  커널  영역  시작까지의  오프셋이  사용  가능한  RAM  크기를  초과하지  않는  가상  주소를  사용한
액세스는  자동으로  물리적  페이지  프레임과  연결됩니다.  메모리  때문에  실용적입니다.

이  체계가  채택되면  커널  영역의  할당은  항상  물리적  RAM에  배치됩니다.
그러나  한  가지  문제가  있습니다.  커널의  가상  주소  공간  부분은  반드시  CPU의  이론상  최대  주소  공간보다  작습니다.
커널  주소  공간에  매핑할  수  있는  것보다  더  많은  물리적  RAM이  있는  경우  커널은  '초과잉''  메모리를  관리하기
위해  highmem  방법을  사용해야  합니다.
IA-32  시스템에서는  최대  896MiB의  RAM을  직접  관리할  수  있습니다.
이  수치를  초과하는  것(최대  4GiB)은  highmem을  통해서만  처리될  수  있습니다.

4GiB  는  32비트  시스템  에서  처리할  수  있는  최대  메모리  크기입니다  (232  =  4GiB ).
트릭을  사용하는  경우  최신  IA-32  구현(Pentium  PRO  이상)은  PAE  모드가  활성화된  경우  최대  64GiB의  메모리를
관리할  수  있습니다.
PAE는  페이지  주소  확장을  나타내며  메모리  포인터에  대한  추가  비트를  제공합니다.
그러나  64GiB  전체  를  동시에  처리할  수는  없으며  각각  4GiB  의  섹션만  처리할  수  있습니다.
대부분의  메모리  관리  데이터  구조는  0  ~  1GiB  범위에서만  할당할  수  있으므로  최대  메모리  크기에는
실질적인  제한이  있으며  이는  64GiB  미만입니다 .
정확한  값은  커널  구성에  따라  다릅니다.  예를  들어,  highmem에  세  번째  수준의  페이지  테이블  항목을  할당하여
일반  영역의  로드를  줄이는  것이  가능합니다.
4GiB  를  초과하는  메모리를  갖춘  IA-32  시스템은  드물고  모든  실용적인  목적을  위해  IA-32를
대체한  64비트  아키텍처  AMD64가  이  문제에  대한  훨씬  깔끔한  솔루션을  제공하므로  두  번째  하이메모리에
대해서는  굳이  논의하지  않겠습니다.  여기  모드.

물리적  주소  지정이  더  작은  수의  비트(예:  48  또는  52)로  제한되어  있더라도  사용  가능한  주소  공간이  방대하기  때문에
64비트  시스템에서는  Highmem  모드가  필요하지  않습니다.  불과  몇  년  전만  해도  32비트  시스템의  주소  공간이
64비트  시스템의  한계에  도달하는  것은  단지  시간  문제인  것처럼  보일  것이라고  주장할  수  있었지만,
당분간은  16  EiB로  충분할  것  입니다 .  하지만  당신은  결코  알지  못합니다 ... .

highmem  페이지  사용은  커널  자체에서만  문제가  됩니다.
커널은  먼저  아래에  설명된  kmap  및  kunmap  함수를  호출하여  highmem  페이지를  사용하기  전에  가상  주소
공간에  매핑해야  합니다.  이는  일반  메모리  페이지에서는  필요하지  않습니다.
그러나  사용자  공간  프로세스의  경우  페이지가  항상  페이지  테이블을  통해  액세스되고  직접적으로  액세스되지
않기  때문에  페이지가  highmem  페이지인지  일반  페이지인지는  전혀  차이가  없습니다.

서로  다른  방식으로  물리적  메모리를  관리하는  두  가지  유형의  머신이  있습니다.

1.  UMA  시스템  (균일한  메모리  액세스)  은  사용  가능한  메모리를  연속적인  방식으로  구성합니다(작은  간격이  있을  수  있음).
대칭형  다중  프로세서  시스템의  각  프로세서는  각  메모리  영역에  동일하게  빠르게  액세스할  수  있습니다.
2.  NUMA  머신  (비균일  메모리  액세스)  은  항상  다중  프로세서  머신입니다.  특히  빠른  액세스를  지원하기  위해  시스템의  각
CPU에  로컬  RAM을  사용할  수  있습니다.  프로세서는  버스를  통해  연결되어  다른  CPU의  로컬  RAM에  대한  액세스를  지원합니다.  이는  당연히  로컬  RAM에  액세스하는  것보다  느립니다.
이러한  시스템의  예로는  IBM의  Alpha  기반  WildFire  서버와  NUMA-Q  시스템이  있습니다.

.. image:: ./img/fig3_1.png


연속되지  않은  메모리를  사용하는  두  머신  유형을  혼합하는  것도  가능합니다.
이러한  혼합은  RAM이  연속적이지  않지만  큰  구멍이  있는  UMA  시스템을  나타냅니다.
여기서는  NUMA  구성의  원칙을  적용하여  커널에  대한  메모리  액세스를  더  간단하게  만드는  것이  도움이  되는  경우가  많습니다.  실제로  커널은  FLATMEM,  DISCONTIGMEM  및  SPARSEMEM의  세  가지  구성  옵션을  구별합니다. SPARSEMEM  과  DISCONTIGMEM은  실질적으로  동일한  목적을  수행하지만  개발자의  관점에서는  코드  품질이  다릅니다.  SPARSEMEM은  더  실험적이고  덜  안정적인  것으로  간주되지만  성능  최적화  기능을  제공합니다.  불연속  메모리는  더  안정적인  것으로  추정되지만  메모리  핫플러그와  같은  새로운  기능에는  준비가  되어  있지  않습니다.
다음  섹션에서는  이  메모리  구성  유형이  대부분의  구성에서  사용되고  일반적으로  커널  기본값이기도  하기  때문에  주로
FLATMEM  으로  제한합니다.  모든  메모리  모델이  실질적으로  동일한  데이터  구조를  사용하기  때문에  다른  옵션을  논의하지
않는다는  사실은  큰  손실이  아닙니다.
실제  NUMA  시스템은  구성  옵션  CONFIG_NUMA를  설정하며  메모리  관리  코드는  두  변형  간에  다릅니다.
플랫  메모리  모델은  NUMA  시스템에서는  적합하지  않으므로  연속되지  않고  희박한  메모리만  사용할  수  있습니다.
구성  옵션  NUMA_EMU를  사용하면  플랫  메모리가  있는  AMD64  시스템이  메모리를  가짜  NUMA  영역으로  분할하여
NUMA  시스템의  전체  복잡성을  누릴  수  있습니다.  이는  실제  NUMA  머신을  사용할  수  없는  경우  개발에  유용할
수  있습니다.  어떤  이유로든  비용이  많이  드는  경향이  있습니다.

이  책은  UMA  사례에  초점을  맞추고  CONFIG_NUMA를  고려하지  않습니다.
이는  NUMA  데이터  구조를  완전히  무시할  수  있다는  의미는  아닙니다.
UMA  시스템은  주소  공간에  큰  홀이  포함된  경우  구성  옵션  CONFIG_DISCONTIGMEM을  선택할  수  있으므로
NUMA  기술을  사용하지  않는  시스템에서도  둘  이상의  메모리  노드를  사용할  수  있습니다.

그림  3-2에는  메모리  레이아웃과  관련된  구성  옵션에  대한  다양한  선택  사항이  요약되어  있습니다.
다음  논의에서  용어  할당  순서를  자주  접하게  될  것입니다 .
메모리  영역에  포함된  페이지  수의  이진  로그를  나타냅니다.
순서  0  할당은  한  페이지,  순서  2  할당은  21  =  2  페이지,  순서  3  할당은  22  =  4  페이지로  구성됩니다.

.. image:: ./img/fig3_2.png




3.2 Organization in the (N)UMA Model
===================================
지원되는  다양한  아키텍처는  메모리  관리  방법에  따라  크게  다릅니다.
커널의  지능적인  설계와  어떤  경우에는  호환성  계층  사이에  있기  때문에  이러한  차이점은  너무  잘  숨겨져  일반
코드에서는  일반적으로  이를  무시할  수  있습니다.
1장에서  설명한  것처럼  주요  문제는  페이지  테이블의  간접  참조  수준  수가  다양하다는  것입니다.
두  번째  핵심  측면은  NUMA와  UMA  시스템으로의  분할입니다.

커널은  균일하고  균일하지  않은  메모리  액세스를  가진  시스템에  대해  동일한  데이터  구조를  사용하므로
개별  알고리즘은  다양한  형태의  메모리  배열을  거의  또는  전혀  구별할  필요가  없습니다.
UMA  시스템에서는  전체  시스템  메모리를  관리하는  데  도움이  되는  단일  NUMA  노드가  도입되었습니다.
메모리  관리의  다른  부분은  의사  NUMA  시스템으로  작업하고  있다고  믿게  됩니다.

3.2.1 Overview
------------------
커널에서  메모리를  구성하는  데  사용되는  데이터  구조를  살펴보기  전에  용어가  항상  이해하기  쉽지  않기  때문에
몇  가지  개념을  정의해야  합니다.  먼저  NUMA  시스템을  고려해  보겠습니다.
이를  통해  UMA  시스템으로  축소하는  것이  매우  쉽다는  것을  보여줄  수  있습니다.
그림  3-3은  아래에  설명된  메모리  분할을  그래픽으로  나타낸  것입니다(데이터  구조를  자세히  살펴보면  알  수  있듯이
상황은  다소  단순화되었습니다).
첫째,  RAM  메모리는  노드로  구분됩니다 .  노드는  시스템의  각  프로세서와  연관되어  있으며  커널에서  pg_data_t  의
인스턴스로  표시됩니다  (이러한  데이터  구조는  곧  정의됩니다).
각  노드는  메모리를  더  세분화하여  영역  으로  분할됩니다 .  예를  들어,  DMA  작업(ISA  장치의  경우)에  사용할
수  있는  메모리  영역에  대한  제한이  있습니다.
처음  16MiB만  적합합니다.  직접  매핑할  수  없는  highmem  영역도  있습니다.
이들  사이에는  보편적으로  사용되는  '일반'  메모리  영역이  있습니다.
따라서  노드는  최대  3개의  영역으로  구성됩니다.
커널은  이들을  구별하기  위해  다음과  같은  상수를  도입합니다.

.. image:: ./img/fig3_3.png


커널은  시스템의  모든  영역을  열거하기  위해  다음  상수를  도입합니다.

<mmzone.h>
enum zone_type {
#ifdef CONFIG_ZONE_DMA
ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
ZONE_DMA32,
#endif
ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
ZONE_HIGHMEM,
#endif
ZONE_MOVABLE,
MAX_NR_ZONES
};

DMA에  적합한  메모리의  경우  ZONE_DMA입니다 .  이  영역의  크기는  프로세서  유형에  따라  다릅니다.  IA-32  시스템에서
제한은  고대  ISA  장치에  의해  부과된  기존  16MiB  경계입니다.
그러나  더  현대적인  기계도  이로  인해  영향을  받을  수  있습니다.

32비트  주소  지정  가능  영역의  DMA  적합  메모리용  ZONE_DMA32 .  분명히  64비트  시스템의  두  DMA  대안  간에는  차이
점만  있습니다.  32비트  시스템에서는  이  영역이  비어  있습니다.  즉,  크기는  0MiB입니다.
예를  들어  Alphas  및  AMD64  시스템에서  이  영역의  범위는  0~4GiB입니다.

커널 세그먼트에  직접  매핑된  일반  메모리의  경우  ZONE_NORMAL입니다 .  이는  모든  아키텍처에  존재할  수  있다고  보장
되는  유일한  영역입니다.  그러나  해당  영역에  메모리가  반드시  장착되어야  한다는  보장은  없습니다.  예를  들어  AMD64  시스템에  2GiB  RAM이  있는
경우  모든  RAM은  ZONE_DMA32  에  속  하고  ZONE_NORMAL은  비어  있습니다.

ZONE_HIGHMEM은  커널  세그먼트  이상으로  확장되는  물리적  메모리용입니다.

컴파일  시간  구성에  따라  일부  영역을  고려할  필요가  없습니다.
예를  들어  64비트  시스템에는  높은  메모리  영역이  필요하지  않으며  DMA32  영역은
최대  4GiB의  메모리에만  액세스할  수  있는  32비트  주변  장치도  지원하는  64비트  시스템에만  필요합니다.

커널은  물리적  메모리의  조각화를  방지하기  위해  노력할  때  필요한  의사  영역  ZONE_MOVABLE을  추가로  정의합니다.
우리는  섹션  3.5.2에서  이  메커니즘을  자세히  살펴볼  것입니다.
MAX_NR_ZONES는  커널이  시스템에  존재하는  모든  영역을  반복하려는  경우  종료  마커  역할을  합니다.
각  영역은  해당  영역에  속하는  물리적  메모리  페이지( 커널에서  페이지  프레임  이라고  함)가  구성되는  배열과  연결됩니다 .
필요한  관리  데이터가  있는  구조체  페이지  의  인스턴스가  각  페이지  프레임에  할당됩니다.
노드는  커널이  노드를  탐색할  수  있도록  단일  연결  리스트에  보관됩니다.
성능상의  이유로  커널은  항상  현재  실행  중인  CPU와  연결된  NUMA  노드에서  프로세스의  메모리  할당을  수행하려고  시도합니다.
그러나  이것이  항상  가능한  것은  아닙니다.  예를  들어  노드가  이미  가득  찼을  수  있습니다.
이러한  상황에서는  각  노드가  대체  목록을  제공합니다(struct  zonelist의  도움으로 ).
목록에는  메모리  할당의  대안으로  사용할  수  있는  다른  노드(및  관련  영역)가  포함되어  있습니다.
목록에서  항목이  뒤로  갈수록  적합성이  떨어지는  것입니다.
UMA  시스템의  상황은  어떻습니까?  여기에는  노드가  하나만  있고  다른  노드는  없습니다.
이  노드는  그림에서  회색  배경으로  표시됩니다.  다른  모든  것은  변경되지  않습니다.

3.2.2 Data Structures
----------------------
이제  메모리  관리에  사용되는  다양한  데이터  구조  간의  관계를  설명했으므로  각각의  정의를  살펴보겠습니다.


Node Management
------------------
pg_data_t  는  노드를  나타내는  데  사용되는  기본  요소이며  다음과  같이  정의됩니다.

<mmzone.h>
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];
    struct zonelist node_zonelists[MAX_ZONELISTS];
    int nr_zones;
    struct page *node_mem_map;
    struct bootmem_data *bdata;
    unsigned long node_start_pfn;
    unsigned long node_present_pages; /* total number of physical pages */
    unsigned long node_spanned_pages; /* total size of physical page
    range, including holes */
    int node_id;
    struct pglist_data *pgdat_next;
    wait_queue_head_t kswapd_wait;
    struct task_struct *kswapd;
    int kswapd_max_order;
} pg_data_t;

node_zones  는  노드에  있는  영역의  데이터  구조를  보유하는  배열입니다.
node_zonelists는  현재  영역에  더  이상  사용  가능한  공간이  없는  경우  메모리  할당에  사용되는
순서대로  대체  노드와  해당  영역을  지정합니다.
노드의  서로  다른  영역  수는  nr_zones에  보관됩니다.
node_mem_map은  메모리의  모든  물리적  페이지를  설명하는  데  사용되는  페이지  인스턴스  배열에  대한  포인터입니다.
마디.  여기에는  노드에  있는  모든  영역의  페이지가  포함됩니다.
시스템  부팅  중에  메모리  관리가  초기화되기  전에도  커널에  메모리가  필요합니다.
초기화됩니다(메모리  관리를  초기화하려면  메모리도  예약되어야  함).
이  문제를  해결하기  위해  커널은  섹션  3.4.3에  설명된  부팅  메모리  할당자를  사용합니다.
bdata는  부팅  메모리  할당자를  특징짓는  데이터  구조의  인스턴스를  가리킵니다.
􀀁  node_start_pfn  은  NUMA  노드의  첫  번째  페이지  프레임의  논리  번호입니다.  그  페이지
시스템에  있는  모든  노드  의  프레임에는  연속적으로  번호가  지정되며  각  프레임에는  노드에만  고유한  것이  아니라
전역적으로  고유한  번호가  지정됩니다.  첫  번째  페이지  프레임이  0인  노드가  하나만  있기  때문
에  node_start_pfn  은  UMA  시스템에서  항상  0입니다.
node_present_pages는  영역의  페이지  프레임  수를  지정하고  node_spanned_pages는  페이지  프레임의  영역  크기를  지정합니다.
실제  페이지  프레임에  의해  지원되지  않는  영역에  구멍이  있을  수  있으므로  이  값은  반드시  node_present_pages  와
동일할  필요는  없습니다 .
node_id  는  전역  노드  식별자입니다.
시스템의  모든  NUMA  노드에는  번호가  매겨져  있습니다.
0부터.
􀀁  pgdat_next는  평소와  같이  끝이  표시된  단일  연결  목록에  시스템의  노드를  연결합니다.
널  포인터로.
􀀁  kswapd_wait는  영역  외부에서  프레임을  교환할  때  필요한  스왑  데몬에  대한  대기  대기열입니다
(이  내용은  18장에서  자세히  다룹니다).
kswapd는  영역을  담당하는  스왑  데몬의  작업  구조를  가리킵니다.  kswapd_max_order는
해제할  영역의  크기를  정의하기  위해  스와핑  하위  시스템의  구현에  사용되며  현재는  관심이  없습니다.

노드와  노드가  포함하는  영역과  그림  3-3에  표시된  대체  목록  간의  연결은  데이터  구조  시작  부분의  배열을  통해  설정됩니다.

이는  배열에  대한  일반적인  포인터가  아닙니다.  배열  데이터는  노드  구조  자체에  보관됩니다.

노드의  영역은  node_zones[MAX_NR_ZONES]에  보관됩니다.
노드에  더  적은  영역이  있는  경우에도  어레이에는  항상  3개의  항목이  있습니다.
후자의  경우  나머지  항목은  null  요소로  채워집니다.

노드  상태  관리  시스템에  노드가  두  개  이상  존재할  수  있는  경우  커널은  각  노드에  대한  상태
정보를  제공하는  비트맵을  유지합니다.
상태는  비트마스크를  사용하여  지정되며  다음  값이  가능합니다.
<nodemask.h>
enum node_states {
    N_POSSIBLE, /* The node could become online at some point */
    N_ONLINE, /* The node is online */
    N_NORMAL_MEMORY, /* The node has regular memory */
#ifdef CONFIG_HIGHMEM
    N_HIGH_MEMORY, /* The node has regular or high memory */
    #else
    N_HIGH_MEMORY = N_NORMAL_MEMORY,
    #endif
    N_CPU, /* The node has one or more cpus */
    NR_NODE_STATES
};

CPU  및  메모리  핫플러깅에는  N_POSSIBLE,  N_ONLINE  및  N_CPU  상태  가  필요하지만이  책에서는  기능을  고려하지  않습니다.
메모리  관리에  필수적인  플래그는  N_HIGH_MEMORY  입니다.
및  N_NORMAL_MEMORY.  첫  번째는  해당  영역에  다음과  같은  메모리가  장착되어  있다고  발표합니다.
일반  메모리이거나  높은  메모리일  수  있습니다.  N_NORMAL_MEMORY  는  높은  메모리가  아닌  메모리가  있는  경우에만  설정됩니다.
노드에서.비트  필드  또는  특정  노드의  비트를  각각  설정하거나  지우기  위해  두  가지  보조  기능이  제공됩니다.

<nodemask.h>
void node_set_state(int node, enum node_states state)
void node_clear_state(int node, enum node_states state)

또한  for_each_node_state(__node,  __state)  매크로는  모든  노드에  대한  반복을  허용합니다.
특정  상태에  있고  for_each_online_node(node)는  모든  활성  노드를  반복합니다.
커널이  단일  노드만  지원하도록  컴파일되면,  즉  플랫  메모리  모델을  사용하면  해당  노드는비트맵이
존재하지  않으며  이를  조작하는  함수는  단순히  수행하는  빈  작업으로  확인됩니다.
아무것도  아님.


Memory Zones
------------------
커널은  zones  구조를  사용하여  영역을  설명합니다.  이는  다음과  같이  정의됩니다:
<mmzone.h>
struct zone {
/* Fields commonly accessed by the page allocator */
unsigned long pages_min, pages_low, pages_high;
unsigned long lowmem_reserve[MAX_NR_ZONES];
struct per_cpu_pageset pageset[NR_CPUS];
/*
* free areas of different sizes
*/
spinlock_t lock;
struct free_area free_area[MAX_ORDER];
ZONE_PADDING(_pad1_)
/* Fields commonly accessed by the page reclaim scanner */
spinlock_t lru_lock;
struct list_head active_list;
struct list_head inactive_list;
unsigned long nr_scan_active;
unsigned long nr_scan_inactive;
unsigned long pages_scanned; /* since last reclaim */
unsigned long flags; /* zone flags, see below */
/* Zone statistics */
atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
int prev_priority;
ZONE_PADDING(_pad2_)
/* Rarely used or read-mostly fields */
wait_queue_head_t * wait_table;
unsigned long wait_table_hash_nr_entries;
unsigned long wait_table_bits;
/* Discontig memory support fields. */
struct pglist_data *zone_pgdat;
unsigned long zone_start_pfn;
unsigned long spanned_pages; /* total size, including holes */
unsigned long present_pages; /* amount of memory (excluding holes) */
/*
* rarely used fields:
*/
char *name;
} ____cacheline_maxaligned_in_smp;

이  구조의  눈에  띄는  측면은  ZONE_PADDING으로  구분된  여러  섹션으로  나누어져  있다는  것입니다 .
이는  영역  구조에  매우  자주  액세스되기  때문입니다.
다중  프로세서  시스템에서는  일반적으로서로  다른  CPU가  동시에  구조  요소에  액세스하려고  시도하는  경우가  발생합니다.
자물쇠(에서  조사됨따라서  5장)은  서로  간섭하고  오류가  발생하는  것을  방지하는  데  사용됩니다.불일치.
구조의  두  스핀록( zone->lock  및  zone->lru_lock )은  종종커널이  구조에  매우  자주  액세스하기  때문에  획득됩니다.1
데이터는  CPU  캐시에  보관되어  더  빠르게  처리됩니다.
캐시는  라인으로  구분되며,각  라인은  다양한  메모리  영역을  담당합니다.
커널은  ZONE_PADDING  매크로를  호출하여각  잠금이  자체  캐시  라인에  있는지  확인하기  위해  구조에  추가되는  '패딩''을  생성합니다.
최적의  캐시를  달성하기  위해  컴파일러  키워드  __cacheline_maxaligned_in_smp  도  사용됩니다.
조정.
구조의  마지막  두  섹션도  패딩으로  서로  분리됩니다.  둘  다  포함하지  않음잠금의  주요  목적은  빠른  액세스를  위해
데이터를  캐시  라인에  유지하여RAM  메모리에서  데이터를  로드해야  하는데  이는  느린  프로세스입니다.
이로  인해  크기가  증가함패딩  구조는  무시할  수  있습니다.
특히  영역  구조의  인스턴스가  상대적으로  적기  때문입니다.
커널  메모리.
구조  요소의  의미는  무엇입니까?  메모리  관리는  커널의  복잡하고  포괄적인  부분이기  때문에
이  시점에서  모든  요소의  정확한  의미를  다루는  것은  불가능합니다.
이  장과  다음  장의  일부는  관련  데이터  구조를  이해하는  데  전념할  것입니다.그리고  메커니즘.
그러나  제가  제공할  수  있는  것은  제가  겪고  있는  문제를  맛보게  해주는  개요입니다.
논의할  예정입니다.  그럼에도  불구하고  많은  수의  전방  참조는  불가피합니다.

페이지가  교체될  때  페이지_분,  페이지_하이  및  페이지_로우가  '워터마크'로  사용됩니다.

사용  가능한  RAM  메모리가  부족한  경우  커널은  하드  디스크에  페이지를  쓸  수  있습니다.
이  세  가지  요소는  스와핑  데몬의  동작에  영향을  미칩니다.
􀀁  page_high  보다  많은  페이지가  사용  가능하면  영역  상태가  이상적입니다.  􀀁  사용  가능한  페이지  수
가  page_low  아래로  떨어지면  커널은  페이지를  다른  페이지로  교체하기  시작합니다.
하드  디스크.
􀀁  사용  가능한  페이지  수가  page_min  미만이면  페이지  회수에  대한  부담이  커집니다.
해당  영역에  여유  페이지가  긴급하게  필요하기  때문에  증가했습니다.
18장에서는  완화를  찾기  위한  커널의  다양한  수단에  대해  논의할  것입니다.
이러한  워터마크의  중요성은  주로  18장에  표시되지만  섹션  3.5.5에서도  적용됩니다.
lowmem_reserve  배열은  어떤  상황에서도  실패해서는  안  되는  중요한  할당을  위해  예약된  각
메모리  영역에  대해  여러  페이지를  지정합니다.
각  영역은  그  중요성에  따라  기여합니다.  개별  기여도를  계산하는  알고리즘은  섹션  3.2.2에서  논의됩니다.

이지  세트  는  CPU별  핫  앤  콜드  페이지  목록을  구현하기  위한  배열입니다.
커널은  이러한  목록을  사용하여  구현을  만족시키는  데  사용할  수  있는  새로운  페이지를  저장합니다.  그
러나  캐시  상태로  구분됩니다.  여전히  캐시  핫  상태일  가능성이  높으므로  빠르게  액세스할  수  있는  페이지는
캐시  콜드  페이지와  구분됩니다.
다음  섹션에서는  이  동작을  실현하는  데  사용되는  struct  per_  cpu_pageset  데이터  구조에  대해  설명합니다.
free_area는  버디  시스템을  구현하는  데  사용되는  동일한  이름의  데이터  구조  배열입니다.
각  배열  요소는  고정된  크기의  연속  메모리  영역을  나타냅니다.  각  영역에  포함된  여유  메모리  페이지  관리는  free_area부터  시작됩니다 .
사용된  데이터  구조는  자체적으로  논의할  가치가  있으며  섹션  3.5.5에서는  버디  시스템의  구현  세부  사항을  심층적으로  다룹니다.

두번째  섹션의  요소는  활동에  따라  영역에서  사용되는  페이지를  분류하는  역할을  합니다.  자주  액세스되는  페이지는  커널에  의해  활성  페
이지로  간주됩니다 .  비활성  페이지는  분명히  그  반대입니다.  이러한  구별은  페이지를  교체해야  할  때  중요합니다.
가능하다면  자주  사용하는  페이지는  그대로  두어야  하지만  불필요한  비활성  페이지는  처벌  없이  교체될  수  있습니다.

다음  요소가  관련됩니다.

tive_list는  활성  페이지를  수집하고  inactive_list는  비활성  페이지  (페이지
인스턴스).
􀀁  nr_scan_active  및  nr_scan_inactive는  메모리를  회수할  때  스캔할  활성  및  비활성  페이지  수를  지정합니다.
􀀁  Pages_scanned는  페이지가  마지막으로  교체된  이후  스캔에  실패한  페성  및  비활성  페이지  수를  지정합니다.
플래그는 영역의 현재 상태를 설명합니다. 다음 플래그가 허용됩니다.
<mmzone.h>
typedef enum {
ZONE_ALL_UNRECLAIMABLE, /* all pages pinned */
ZONE_RECLAIM_LOCKED, /* prevents concurrent reclaim */
ZONE_OOM_LOCKED, /* zone is in OOM killer zonelist */
} zone_flags_t;
또한  이러한  플래그  중  어느  것도  설정되지  않을  수도  있습니다.  이는  해당  영역의  정상적인  상태입니다.
ZONE_ALL_UNRECLAIMABLE  은  커널이  존의  일부  페이지를  재사용하려고  할  때  (페이지  회수,  18장  참조)
발생할  수  있는  상태이지만  모든  페이지가  고정되어  있기  때문에  전혀  가능하지  않습니다 .
예를  들어,  사용자  공간  애플리케이션은  mlock  시스템  호출을  사용하여  페이지를  교체하는  등의  방법으로
페이지를  물리적  메모리에서  제거해서는  안  된다는  것을  커널에  지시할  수  있습니다.
이러한  페이지가  고정되어  있다고  합니다.  영역의  모든  페이지에  이러한  문제가  발생하면  영역을  회수할  수  없으며
플래그가  설정됩니다.  시간을  낭비하지  않기  위해  스와핑  데몬은  회수할  페이지를  찾을  때  이러한  종류의  영역을
아주  잠깐  스캔합니다.
SMP  시스템에서는  여러  CPU가  동시에  영역을  회수하려는  유혹을  받을  수  있습니다.
ZONE_RECLAIM_LOCKED  플래그는  이를  방지합니다.  CPU가  영역을  회수하는  경우  플래그를  설정합니다.
이렇게  하면  다른  CPU가  시도하는  것을  방지할  수  있습니다.
ZONE_OOM_LOCKED  는  불행한  상황을  위해  예약되어  있습니다.
프로세스가  너무  많은  메모리를  사용하여  필수  작업을  더  이상  완료할  수  없는  경우  커널은  더  많은  여유
페이지를  얻기  위해  최악의  메모리  먹는  사람을  선택하고  이를  종료하려고  시도합니다.
이  플래그는  이  경우  여러  CPU가  방해가  되는  것을  방지합니다.
커널은  영역  플래그를  테스트하고  설정하기  위한  세  가지  보조  기능을  제공합니다.
<mmzone.h>
void zone_set_flag(struct zone *zone, zone_flags_t flag)
int zone_test_and_set_flag(struct zone *zone, zone_flags_t flag)
void zone_clear_flag(struct zone *zone, zone_flags_t flag)

zone_set_flag  및  zone_clear_flag는  각각  특정  플래그를  설정하고  삭제합니다.
zone_test_  and_set_flag는  먼저  주어진  플래그가  설정되어  있는지  테스트하고  그렇지  않으면  그렇게  합니다.
플래그의  이전  상태가  호출자에게  반환됩니다.

vm_stat는  영역에  대한  수많은  통계  정보를  보관합니다.
여기에  보관된  대부분의  정보는  현재로서는  그다지  의미가  없으므로  자세한  논의는  섹션  17.7.1에
서  연기됩니다.  지금은  정보가  커널  전체에서  업데이트된다는  점만  알아도  충분합니다.  보
조  함수  zone_page_state를  사용하면  vm_stat  의  정보를  읽을  수  있습니다 .
<vmstat.h>
static inline unsigned long zone_page_state(struct zone *zone,
enum zone_stat_item item)

예를  들어  항목은  위에서  설명한  active_list  및  inactive_list  에  저장된  활성  및  비활성  페이지  수를
쿼리하기  위해  NR_ACTIVE  또는  NR_INACTIVE  일  수  있습니다.
영역의  사용  가능한  페이지  수는  NR_FREE_PAGES를  통해  얻습니다.

prev_priority는  try_to_free_pages  에서  충분한  페이지  프레임이  해제될  때까지  마지막  검색  작업에서  영역을  검
색한  우선  순위를  저장합니다  (17장  참조).
17장에서도  볼  수  있듯이  매핑된  페이지를  교체할지  여부에  대한  결정은  이  값에  따라  달라집니다.

wait_table,  wait_table_bits  및  wait_table_hash_nr_entries는  페이지가  사용  가능해질  때까지  기다리는  프로
세스에  대한  대기  대기열을  구현합니다.
이  메커니즘의  세부  사항은  14장에  나와  있지만  직관적인  개념은  꽤  잘  적용됩니다.
즉,  프로세스는  어떤  조건을  기다리기  위해  줄을  서서  대기합니다.
이  조건이  true가  되면  커널로부터  알림을  받고  작업을  재개할  수  있습니다.

영역과  상위  노드  간의  연결은  zone_pgdat  에  의해  설정  됩니다.
pg_list_data  의  해당  인스턴스에 .
􀀁  zone_start_pfn  은  해당  영역의  첫  번째  페이지  프레임  인덱스입니다.
􀀁  나머지  세  필드는  거의  사용되지  않으므로  데이터  구조의  끝에  배치되었습니다.
name  은  영역의  일반적인  이름을  포함하는  문자열입니다.
현재  Normal,  DMA  및  HighMem의  세  가지  옵션을  사용할  수  있습니다.
spanned_pages는  영역의  총  페이지  수를  지정합니다.
그러나  이미  언급한  것처럼  영역에  작은  구멍이  있을  수  있으므로  모두  사용할  필요는  없습니다.
따라서  추가  카운터  (present_pages)  는  실제로  사용  가능한  페이지  수를  나타냅니다.
일반적으로  이  카운터의  값은spanned_pages의  값과  동일합니다 .

Calculation of Zone Watermarks
------------------------------
다양한  워터마크를  계산하기  전에  커널은  먼저  중요한  할당을  위해  사용  가능한  최소  메모리  공간을  결정합니다.
이  값은  사용  가능한  RAM  크기에  따라  비선형적으로  확장됩니다.  전역  변수  min_free_kbytes에  저장됩니다 .
그림  3-4는  스케일링  동작에  대한  개요를  제공하며,  메인  그래프와  달리  메인  메모리  크기에  대수  스케일을
사용하지  않는  삽입은  최대  4GiB까지  영역의  확대를  보여줍니다.  데스크탑  환경에서  일반적으로  사용되는  적당한  메모리를  갖춘  시스템의  상황에  대한  느낌을  제공하기  위한  몇  가지  예시적인  값은  표  3-1에  수집되어  있습니다.  128KiB  이상  64MiB  이하를  사용할  수  있다는  점은  변함이  없습니다.
그러나  상한은  매우  만족스러운  양의  주  메모리가  장착된  시스템에서만  필요하다는  점  에  유의하십시오.
3 /proc/sys/vm/min_free_kbytes  파일을  사용  하면  userland에서  값을  읽고  적용할  수  있습니다.
데이터  구조의  워터마크  채우기는  커널  부팅  중에  호출되며  명시적으로  시작할  필요가  없는  init_per_zone_pages_min  에
의해  처리됩니다.
4 setup_per_zone_pages_min은  구조체  영역  의  페이지_분,  페이지_로우  및  페이지_하이  요소를  설정합니다 .
highmem  영역  외부의  총  페이지  수가  계산된  후(그리고  lowmem_  페이지에  저장됨)  커널은  시스템의
모든  영역을  반복하고  다음  계산을  수행합니다.

mm/page_alloc.c
void setup_per_zone_pages_min(void)
{
unsigned long pages_min = min_free_kbytes >> (PAGE_SHIFT - 10);
unsigned long lowmem_pages = 0;
struct zone *zone;
unsigned long flags;
...
for_each_zone(zone) {
u64 tmp;
tmp = (u64)pages_min * zone->present_pages;
do_div(tmp,lowmem_pages);
if (is_highmem(zone)) {
int min_pages;
min_pages = zone->present_pages / 1024;
if (min_pages < SWAP_CLUSTER_MAX)
min_pages = SWAP_CLUSTER_MAX;
if (min_pages > 128)
min_pages = 128;
zone->pages_min = min_pages;
} else {
zone->pages_min = tmp;
}
zone->pages_low = zone->pages_min + (tmp >> 2);
zone->pages_high = zone->pages_min + (tmp >> 1);
}
}

.. image:: ./img/fig3_4.png

.. image:: ./img/fig3_5.png

highmem  영역의  하한인  SWAP_CLUSTER_MAX는  전체  페이지에서  중요한  수량입니다. 17장에서  설명한  대로  하위  시스템을
회수합니다.
거기의  코드는  페이지  클러스터에서  배치  방식으로  작동하는  경우가  많습니다. SWAP_CLUSTER_MAX  는
이러한  클러스터의  크기를  정의합니다.
그림  3-4는  다양한  주  메모리  크기에  대한  계산  결과를  보여줍니다.
요즘에는  높은  메모리가  더  이상  중요하지  않기  때문에(대부분 대용량  RAM이  있는  컴퓨터는  64비트  CPU를  사용하므로
결과를  표시하기  위해  그래프를  제한했습니다.
일반  구역의  경우.
lowmem_reserve  계산은  setup_per_zone_lowmem_reserve  에서  수행됩니다 .
커널은  모든  것을  반복합니다.시스템의  노드를  총계로  나누어  노드의  각  구역에  대한  최소  보유량을  계산합니다.
sysctl_lowmem_reserve_ratio[zone]  으로  영역의  페이지  프레임  수 .
기본  설정은제수는  낮은  메모리의  경우  256이고  높은  메모리의  경우  32입니다.

Hot-N-Cold Pages
------------------

구조체  영역  의  페이지  세트  요소는  핫  앤  콜드  할당자를  구현하는  데  사용됩니다.  커널은  다음을  가리킨다.
CPU  캐시에  있는  경우  메모리의  페이지가  핫  하므로  해당  페이지의  데이터에  더  빠르게  액세스할  수  있습니다.
RAM에  있었어요.  반대로  콜드  페이지는  캐시에  보관되지  않습니다.
각  CPU에는  다중  프로세서  시스템에  하나  이상의  캐시가  있으므로  각  CPU에  대해  관리가  별도로  이루어져야  합니다.

영역이  특정  NUMA  노드에  속하고  따라서  특정  CPU와  연결되어  있더라도  다른  CPU의  캐시에는  이  영역의  페이지가  포함될  수  있습니다.
궁극적으로  각  프로세서는  속도는  다르지만  시스템의  모든  페이지에  액세스할  수  있습니다.
따라서  영역별  데이터  구조는  해당  영역의  NUMA  노드와  연결된  CPU뿐만  아니라  해당  영역의
다른  모든  CPU에도  적합해야  합니다.

페이지  세트  는  시스템이  수용할  수  있는  최대  CPU  수만큼  많은  항목을  보유하는  배열입니다.
<mmzone.h>
struct zone {
...
struct per_cpu_pageset pageset[NR_CPUS];
...
};

NR_CPUS  는  컴파일  타임에  정의되는  구성  가능한  전처리기  상수입니다.
이  값은  단일  프로세서  시스템에서는  항상  1이지만  SMP  시스템용으로  컴파일된  커널에서는
2에서  32  사이(또는  64비트  시스템에서는  64)일  수  있습니다.

이  값은  시스템에  실제로  존재하는  CPU  수가  아니라  커널이  지원하는  최대  CPU  수를  반영합니다.

배열  요소는  다음과  같이  정의된  per_cpu_pageset  유형입니다.

<mmzone.h>
struct per_cpu_pageset {
struct per_cpu_pages pcp[2]; /* 0: hot. 1: cold */
} ____cacheline_aligned_in_smp;


구조는  두  개의  항목이  있는  배열로  구성됩니다.
첫  번째는  핫  페이지를  관리하고  두  번째는  콜드  페이지를  관리합니다.
유용한  데이터는  per_cpu_pages에  보관됩니다


<mmzone.h>
struct per_cpu_pages {
int count; /* number of pages in the list */
int high; /* high watermark, emptying needed */
int batch; /* chunk size for buddy add/remove */
struct list_head list; /* the list of pages */
};

count는  요소와  관련된  페이지  수를  기록하는  반면  high  는  워터마크입니다.
count  값이  high를  초과  하면  목록에  페이지가  너무  많다는  의미입니다.
낮은  채우기  상태에는  명시적인  워터마크가  사용되지  않습니다.
요소가  남아  있지  않으면  목록이  다시  채워집니다.
list는  CPU당  페이지를  보유하고  커널의  표준  방법을  사용하여  처리되는  이중  연결  목록입니다.
가능하다면  CPU당  캐시는  개별  페이지로  채워지지  않고  여러  페이지  청크로  채워집니다.
배치는  단일  패스에  추가할  페이지  수에  대한  지침입니다.
그림  3-6은  듀얼  프로세서  시스템에서  CPU당  캐시의  데이터  구조가  어떻게  채워지는지  그래픽으로  보여줍니다.


.. image:: ./img/fig3_6.png




Data Structures
------------------



Creating and Manipulating Entries
-----------------------------------


3.3 Initialization ofMemoryManagement
=======================================


Data Structure Setup
----------------------



Architecture-Specific Setup
----------------------------


3.4 Memory Management during the Boot Process
================================================



3.5 Management of Physical Memory
================================



Structure of the Buddy System
--------------------------------


Avoiding Fragmentation
---------------------------


Initializing the Zone and Node Data Structures
------------------------------------------------


Allocator API
------------------


Reserving Pages
---------------------


Freeing Pages
--------------------


Allocation of Discontiguous Pages in the Kernel
--------------------------------------------------


Kernel Mappings
---------------------



3.6 The Slab Allocator
=========================


Alternative Allocators
---------------------------


Memory Management in the Kernel
--------------------------------


The Principle of Slab Allocation
---------------------------------


Implementation
------------------------


General Caches
-----------------------


3.7 Processor Cache and TLB Control
==================================


Summary
===========