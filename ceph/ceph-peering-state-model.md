**描述"ceph peering state model"的dot文件**

<pre name="code" class="dot"> 
digraph G {
    size="7,7" compound=true;
    subgraph cluster0 {
        label = "RecoveryMachine";
        color = "blue";
        Crashed;
        Initial[shape=Mdiamond];
        Reset;
        subgraph cluster1 {
            label = "Started";
            color = "blue";
            Start[shape=Mdiamond];
            subgraph cluster2 {
                label = "Primary";
                color = "blue";
                WaitActingChange;
                Incomplete;
                subgraph cluster3 {
                    label = "Peering";
                    color = "blue";
                    GetInfo[shape=Mdiamond];
                    GetLog;
                    GetMissing;
                    WaitUpThru;
                }
                Active;
            }
            ReplicaActive;
            Stray;
        }
    }
    Initial -> Reset [label="Load",];
    GetInfo -> WaitActingChange [label="NeedActingChange",ltail=cluster2,];
    GetMissing -> WaitUpThru [label="NeedUpThru",];
    GetInfo -> Active [label="Activate",ltail=cluster3,];
    Stray -> ReplicaActive [label="Activate",];
    Initial -> GetInfo [label="MNotifyRec",lhead=cluster2,];
    Initial -> Stray [label="MLogRec",];
    GetInfo -> GetLog [label="GotInfo",];
    Start -> GetInfo [label="MakePrimary",lhead=cluster2,];
    Reset -> Start [label="ActMap",lhead=cluster1,];
    Start -> Reset [label="AdvMap",ltail=cluster1,];
    GetInfo -> Reset [label="AdvMap",ltail=cluster3,];
    WaitActingChange -> Reset [label="AdvMap",];
    Start -> Stray [label="MakeStray",];
    Initial -> Crashed [label="boost::statechart::event_base",];
    Reset -> Crashed [label="boost::statechart::event_base",];
    Start -> Crashed [label="boost::statechart::event_base",ltail=cluster1,];
    Initial -> Start [label="Initialize",lhead=cluster1,];
    Initial -> Stray [label="MInfoRec",];
    GetInfo -> Incomplete [label="IsIncomplete",ltail=cluster2,];
    GetLog -> GetMissing [label="GotLog",];
}
</pre>


**生成图片**

把上面内容复制到peering.dot文件中
    #sudo apt-get install graphviz
    #dot -Tsvg peering.dot -o peering.svg

打开peering.svg图片，就可以看到peering的状态机了。

![peering](http://way4ever.com/wp-content/uploads/2013/04/ceph-peering-state-model.jpg)

