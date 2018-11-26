## 合约部署报告
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
    currentVote = Vote({okNum:1,voteNum:0,totalNum:memberNum-2,pay:payment,theNode:node,voteResult:VoteResult.NONE,voteType:VoteType.REMOVE});
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
    else if(currentVote.okNum + currentVote.totalNum / 2 < currentVote.voteNum){
        resultOfNO();   //赞成数无法过半，投票不通过
    }
}
```

(7) 投票结果的逻辑，包括通过与不通过两种，为internal类型

```js

```
