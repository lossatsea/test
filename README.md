# 合约部署报告
16340189 邵梓硕

---

### 合约内容

#### 1. 结构体

结构体Member对应所有可能节点，结构体Vote表示一次投票过程。

```js
/*节点*/
struct Member{
    uint weight;  //成员权重，1表示为企划成员，0表示不是企划成员
    uint index;   //成员索引，0表示不是企划成员，非0为成员在成员列表中的索引
}
/*投票*/
struct Vote{
    uint okNum;   //当前投票的赞成数
    uint voteNum; //当前投票的投票数
    uint totalNum;  //当前投票的理论总票数
    uint pay;   //支付款，需要有支付出现时为支付款额，没有支付出现时为0
    address theNode;  //投票针对的节点
    VoteResult voteResult;  //投票结果
    VoteType voteType;  //投票的类型，有“加入”与“移出”两种
    mapping(address => bool) isVoted; //记录节点是否已经投票的映射
}

```

#### 2. 成员变量

在这个阶段简化共享的数据为一个string，在后期会丰富这个数据结构。

```js
uint memberNum = 0; //企划成员数
string info;  //共享的企划信息

/*枚举类型*/
enum VoteState {isVoting, notVoting, none}  //投票状态：正在投票，没有投票，初始化
enum VoteResult {YES, NO, NONE} //投票结果：通过，不通过，初始化
enum VoteType {ADD, REMOVE} //投票类型：添加成员，移出成员

VoteState voteState;  //当前投票状态
Vote currentVote;   //当前的投票

mapping(address => Member) members; //可能节点的映射
mapping(address => uint) debts; //债务单，映射每个可能节点对于这个企划的债务
address[] memberAddr; //成员列表，保存成员的地址
```

#### 3. 修饰器

有很多函数都有相同的require条件，因此为了方便起见编写修饰器。

```js
/*要求这个合约必须先初始化一个企划信息*/
modifier mustInitial{
    require(memberNum > 0, "Project has not been inited.");
    _;
}
/*要求调用者必须是这个企划的成员*/
modifier onlyMember{
    require(members[msg.sender].weight > 0, "You is not the member of project");
    _;
}
```

#### 4. 函数成员

(1) 构造函数

```js
constructor() public payable{
      require(memberNum == 0, "Project has been built.");
  }
```

(2) 初始化企划信息：传入要初始化的企划信息initInfo，在合约构造后使用并且只能使用一次，调用者会自动成为企划的第一个成员。

```js
function initProject(string memory initInfo) public{
    require(memberNum == 0, "Project has been inited.");
    members[msg.sender].weight = 1;
    info = initInfo;  //初始化企划共享信息
    memberNum = 1;
    voteState = VoteState.none;
    memberAddr.push(msg.sender);  //加入成员列表
    members[msg.sender].index = memberNum;  //在成员列表中的索引为成员数
}
```

(3) 加入企划：非企划成员申请加入企划。会发起一个“加入”类型的投票。

```js
function addMember() public mustInitial{
  /*要求调用者不是企划成员并且当前没有正在进行的投票*/
    require(members[msg.sender].weight == 0, "You have been the member of project.");
    require(voteState != VoteState.isVoting, "Now a voting is going, please wait a minute.");
    
  /*发起一个投票*/
    voteState = VoteState.isVoting;
    currentVote = Vote({okNum:0,voteNum:0,totalNum:memberNum,pay:0,theNode:msg.sender,voteResult:VoteResult.NONE,voteType:VoteType.ADD});
  /*清空成员的投票状态*/
    clearVoted();
}
```

(4) 移出企划：企划成员要求移出另一位成员。传入移出成员的地址node，移出成员需要支付的违约金payment，会发起一个“移出”类型的投票

```js
function removeMember(address node, uint payment) public mustInitial onlyMember{
  /*要求移出者为企划成员并且当前没有正在进行的投票*/
    require(members[node].weight > 0, "This node is not the member of project");
    require(voteState == VoteState.notVoting, "There is a vote going.");
    
  /*发起一个投票*/
    voteState = VoteState.isVoting;
    currentVote = Vote({okNum:0,voteNum:0,totalNum:memberNum-1,pay:payment,theNode:node,voteResult:VoteResult.NONE,voteType:VoteType.REMOVE});
  /*清空成员的投票状态*/
    clearVoted();
  /*默认提出者投赞成票*/
    vote(true);
}
```

(5) 清空成员的投票状态：将成员的投票状态置为false，在发起新投票时使用，为internal类型。

```js
function clearVoted() internal{
  /*遍历成员列表得到成员地址，根据地址更改isVoted*/
    for(uint i = 0; i < memberAddr.length; i++){
        currentVote.isVoted[memberAddr[i]] = false;
    }
}
```

(6) 投票：企划成员对当前发起发投票进行投票，传入布尔值赞成或不赞成。

```js
function vote(bool isOk) public mustInitial onlyMember{
  /*要求当前有正在进行的投票“/
    require(voteState == VoteState.isVoting, "There is no vote going.");
  /*要求调用者不是投票针对的节点*/
    require(currentVote.theNode != msg.sender, "You is the one be voted.");
  /*要求还没有投过票*/
    require(!currentVote.isVoted[msg.sender], "You have voted.");
    
  /*投票*/
    currentVote.isVoted[msg.sender] = true;
    if(isOk) currentVote.okNum++;
    currentVote.voteNum++;
  /*判断当前投票是否结束*/
    if(currentVote.okNum > currentVote.totalNum / 2){
        resultOfOK(); //赞成数过半，投票通过
    }
    else if(currentVote.okNum + currentVote.totalNum / 2 <= currentVote.voteNum){
        resultOfNO();   //赞成数无法过半，投票不通过
    }
}
```

(7) 投票结果的逻辑，包括通过与不通过两种，为internal类型

```js
/*投票通过*/
function resultOfOK() internal{
    voteState = VoteState.notVoting;    //投票状态置为无投票
  /*当投票类型是加入时，将投票针对的节点加入企划之中*/
    if(currentVote.voteType == VoteType.ADD){
        members[currentVote.theNode].weight = 1;
        currentVote.voteResult = VoteResult.YES;    //投票结果为OK
        memberNum++;
        memberAddr.push(currentVote.theNode);
        members[currentVote.theNode].index = memberNum;
    }else{
   /*当投票类型时移出时，将投票针对的节点移出企划，并将罚款计入债务单*/
        members[currentVote.theNode].weight = 0;
        currentVote.voteResult = VoteResult.YES;
        memberNum--;
        delete memberAddr[members[currentVote.theNode].index - 1];
        members[currentVote.theNode].index = 0;
        debts[currentVote.theNode] = currentVote.pay;
    }
}

/*投票不通过*/
function resultOfNO() internal{
    voteState = VoteState.notVoting;    //投票状态置为五投票
    currentVote.voteResult = VoteResult.NO; //投票结果为NO
}
```

(8) 支付罚款：移出的节点需要支付债务单上的债务（支付多了也不会还给他，2333），是payable类型

```js
function pay() public payable mustInitial{
  /*要求调用者在企划的债务单上有债务*/
    require(debts[msg.sender] > 0, "You have no debet in the project.");
    
    debts[msg.sender] -= msg.value;
    if(debts[msg.sender] < 0) debts[msg.sender] = 0;
}
```

(9) 查看投票结果：投票结束后，查看上一次投票的结果，返回投票的类型，结果和针对的节点地址

```js
function checkVoteResult() public view mustInitial returns(VoteType votetype, bool res, address theNode){
  /*要求调用者是企划成员或者就是投票针对的节点自己*/
    require(currentVote.theNode == msg.sender || members[msg.sender].weight > 0, "You have no right to check result.");
  /*要求有历史投票记录，也就是说不能没有投过票*/
    require(voteState != VoteState.none,"There is no vote in the history.");
  /*要求当前没有正在进行的投票*/
    require(voteState == VoteState.notVoting,"The voting is going.");
    
    votetype = currentVote.voteType;
    if(currentVote.voteResult == VoteResult.YES){
        res = true;
    } 
    else {
        res = false;
    }
    return (votetype, res, currentVote.theNode);
}
```

(10) 查询企划信息

```js
/*返回字符串的查询*/
function getInfoOfString() public view mustInitial onlyMember returns(string memory){
    return info;
}

/*返回bytes32的查询*/
function getInfoOfBytes32() public view mustInitial onlyMember returns(bytes32){
    bytes memory information = bytes(info);
    bytes32 res;
    assembly {
        res := mload(add(information, 32))
    }
    return res;
}
```

(11) 更新企划信息：在原信息的基础上增加企划信息，传入新信息字符串newInfo

```js
function updateInfo(string memory newInfo) public mustInitial onlyMember{
    bytes memory _info = bytes(info);
    bytes memory _newInfo = bytes(newInfo);
    bytes memory res = new bytes(_info.length + _newInfo.length);
    for(uint i = 0; i < res.length; i++){
        if(i < _info.length) res[i] = _info[i];
        else res[i] = _newInfo[i - _info.length];
    }
    info = string(res);
} 
```

(12) 查询企划余额

```js
 function getSaving() public view mustInitial onlyMember returns(uint){
    return address(this).balance;
}
```

(13) 查询自己的债务

```js
function getDebt() public view mustInitial returns(uint){
    return debts[msg.sender];
}
```

---

### 部署测试

在remix上进行部署测试：

Deploy合约

![deploy]()

尝试查询信息，抛出合约还未初始化的异常.

![error1]()

当前账户为0xca35b7d915458ef540ade6068dfe2f44e8fa733c，称之为账户1，账户1是企划发起人，让他进行初始化“OK!”，成为第一个成员：

![account1]()

![init]()
![initRes]()

账户1查询合约信息，返回字符串为“OK!”，返回的bytes32为“OK!”的ascll码：

![getInfo1]()
![getInfo2]()

账户1查询合约余额为0，没办法毕竟刚开始：

![getSaving1]()

切换到另一个账户0x14723a09acff6d2a60dcdf7aa4aff308fddc160c，称之为账户2，让他查询合约信息，抛出他不是成员的异常：

![account2]()
![error2]()

现在账户2不是成员什么事情都干不了，他想要加入这个企划，因此调用addMember方法，看到成功调用：

![addMember]()

这时企划内部发起了一次投票，怎么能看得出来？可以让账户2看一下投票结果：

![error3]()

发现异常为“投票正在进行中”，不能查询。此时企划内部这开了一场“激烈”的投票，作为唯一成员的账户1决定投出赞成的一票：

![vote1]()

![voteRes1]()

现在账户2可以去看结果了：

![checkVote1]()

这里的返回值，第一个0表示投票类型是ADD，即“加入”投票，第二个true是结果，投票通过，第三个是本次投片针对的账户，这里就是账户2。现在账户2成为了企划的一员，可以查询企划信息了：

![getInfo3]()

账户2决定做出自己的贡献，他们商议了企划的类型为动画企划，账户2要更新企划信息：

![updateInfo]()
![updateInfoRes]

账户1要看看企划信息是不是进行了更新，调用getIbfoOfString来查看：

![getInfo4]()

没错，确实已经更新了。

切换到第三个账户0x4b0897b0513fdc7c541b6d9d7e929c4e5364d2db，称之为账户3：

![account3]()

他想要加入这个企划，和账户2商量，结果账户2误操作，自己调用了addMember：

![error4]()

抛出了“你已经是成员”的异常，账户3明白要自己去申请，于是调用addMember，企划内又开始了一场投票，账户2投了赞成票:

![vote2]()

但是账户1不看好他，投了否定票，于是投票结果没有超过半数，是不通过：

![vote3]()

![checkVote2]()

这里的返回值第二个是false，说明投票没有通过。

结果经过软磨硬泡账户1勉强答应，账户3终于进入了企划。然而进入企划不久账户3就做出了违规行为，账户1发起了针对账户3移出企划的投票，并设置违约金罚款为20：

![remove]()

账户1太过生气，发起投票后又投了赞成票：

![error5]()

他这才先起来发起投票之后就自动给他计了1票，不过这下账户2也保不了账户3了，也投了赞成票，全票通过，账户3查询看到了结果：

![checkVote3]()

这里的返回值第一个为1，说明投票类型是“移出”投票。

发现自己被逐出了企划，还背了20的债，现在的他查询不了企划的信息：

![getInfo5]()

![getDebtRes1]()

没办法，企划成员亮出债务单随时都可以来要债，只好乖乖把债还了，因为觉得对不起引荐的账户2，决定还30的债务：

![pay]()

![payRes]()

至少他现在看自己的债务，无债一身轻了：

![getDebrRes2]()

这时账户1查询企划的余额，发现从0变成了30：

![getSaving3]()

![getSaving2]()

企划这才有了第一笔金，虽然是罚款，但也算是开始起步了。
