​
环境检查
首先，我们需要确保PHP版本符合要求：

<?php
if (version_compare(PHP_VERSION, '7.4.0', '<')) {
    die('require PHP >= 7.4 !');
}
include("./includes/common.php");

这里的代码确保PHP版本不低于7.4，并引入了common.php文件，它包含了一些通用的配置和函数。

处理文档请求
if(isset($_GET['doc'])){
    $doc = trim($_GET['doc']);
    if(!$conf['apiurl']) $conf['apiurl'] = $siteurl;
    $loadfile = \lib\Template::loadDoc($doc);
    include $loadfile;
    exit;
}
当请求包含doc参数时，系统会加载相应的文档模板并显示。

模块选择
$mod = isset($_GET['mod']) ? $_GET['mod'] : 'index';
系统根据请求参数mod确定要加载的模块，如果没有指定则默认加载首页模块。

邀请码处理
if(isset($_GET['invite'])){
    $invite_code = trim($_GET['invite']);
    $uid = get_invite_uid($invite_code);
    if($uid && is_numeric($uid)){
        $_SESSION['invite_uid'] = intval($uid);
    }
}
如果请求包含invite参数，系统会处理邀请码并将其存储在会话中。

首页处理
if($mod == 'index'){
    if($conf['homepage'] == 2){
        echo '<html><frameset framespacing="0" border="0" rows="0" frameborder="0">
        <frame name="main" src="'.$conf['homepage_url'].'" scrolling="auto" noresize>
    </frameset></html>';
        exit;
    } elseif($conf['homepage'] == 1){
        exit("<script language='javascript'>window.location.href='/user/';</script>");
    }
}
根据配置，系统会显示不同的首页内容。

加载模块模板
$loadfile = \lib\Template::load($mod);
include $loadfile;
系统加载并包含指定模块的模板文件。

API 初始化
<?php
$nosession = true;
define('API_INIT', true);
require './includes/common.php';

if(isset($_GET['s'])){
    \lib\ApiHelper::load_api($_GET['s']);
    exit;
}
这里初始化了API，并根据请求参数load_api来加载相应的API模块。

API 功能实现
查询功能
$act = isset($_GET['act']) ? daddslashes($_GET['act']) : null;
@header('Content-Type: application/json; charset=UTF-8');
if($act == 'query'){
    $pid = intval($_GET['pid']);
    $key = daddslashes($_GET['key']);
    $userrow = $DB->getRow("SELECT * FROM pre_user WHERE uid='{$pid}' limit 1");
    if(!$userrow) exit(json_encode(['code' => -3, 'msg' => '商户ID不存在']));
    if($key !== $userrow['key']) exit(json_encode(['code' => -3, 'msg' => '商户密钥错误']));
    if($userrow['keytype'] == 1) exit(json_encode(['code' => -3, 'msg' => '该商户只能使用RSA签名类型']));

    $orders = $DB->getColumn("SELECT count(*) from pre_order WHERE uid={$pid}");
    $lastday = date("Y-m-d", strtotime("-1 day"));
    $today = date("Y-m-d");
    $order_today = $DB->getColumn("SELECT count(*) from pre_order where uid={$pid} and status=1 and date='$today'");
    $order_lastday = $DB->getColumn("SELECT count(*) from pre_order where uid={$pid} and status=1 and date='$lastday'");

    $result = array(
        "code" => 1,
        "pid" => $pid,
        "key" => $key,
        "active" => $userrow['status'],
        "money" => $userrow['money'],
        "type" => $userrow['settle_id'],
        "account" => $userrow['account'],
        "username" => $userrow['username'],
        "orders" => $orders,
        "orders_today" => $order_today,
        "orders_lastday" => $order_lastday
    );
    exit(json_encode($result));
}

结算记录查询

elseif($act == 'settle'){
    $pid = intval($_GET['pid']);
    $key = daddslashes($_GET['key']);
    $limit = isset($_GET['limit']) ? intval($_GET['limit']) : 10;
    $offset = isset($_GET['offset']) ? intval($_GET['offset']) : 0;
    if($limit > 50) $limit = 50;
    $userrow = $DB->getRow("SELECT * FROM pre_user WHERE uid='{$pid}' limit 1");
    if(!$userrow) exit(json_encode(['code' => -3, 'msg' => '商户ID不存在']));
    if($key !== $userrow['key']) exit(json_encode(['code' => -3, 'msg' => '商户密钥错误']));
    if($userrow['keytype'] == 1) exit(json_encode(['code' => -3, 'msg' => '该商户只能使用RSA签名类型']));

    $rs = $DB->query("SELECT * FROM pre_settle WHERE uid='{$pid}' order by id desc limit {$offset}, {$limit}");
    while($row = $rs->fetch(PDO::FETCH_ASSOC)){
        $data[] = $row;
    }
    if($rs){
        $result = array("code" => 1, "msg" => "查询结算记录成功！", "data" => $data);
    } else {
        $result = array("code" => -1, "msg" => "查询结算记录失败！");
    }
    exit(json_encode($result));
}

订单查询

elseif($act == 'order'){
    if(isset($_GET['sign']) && isset($_GET['trade_no'])){
        $trade_no = daddslashes($_GET['trade_no']);
        if(empty($_GET['sign']) || md5(SYS_KEY.$trade_no.SYS_KEY) !== $_GET['sign']) exit(json_encode(['code' => -3, 'msg' => 'verify sign failed']));
        $row = $DB->getRow("SELECT * FROM pre_order WHERE trade_no='{$trade_no}' limit 1");
    } else {
        $pid = intval($_GET['pid']);
        $key = daddslashes($_GET['key']);
        $userrow = $DB->getRow("SELECT * FROM pre_user WHERE uid='{$pid}' limit 1");
        if(!$userrow) exit(json_encode(['code' => -3, 'msg' => '商户ID不存在']));
        if($key !== $userrow['key']) exit(json_encode(['code' => -3, 'msg' => '商户密钥错误']));
        if($userrow['keytype'] == 1) exit(json_encode(['code' => -3, 'msg' => '该商户只能使用RSA签名类型']));

        if(!empty($_GET['trade_no'])){
            $trade_no = daddslashes($_GET['trade_no']);
            $row = $DB->getRow("SELECT * FROM pre_order WHERE uid='{$pid}' and trade_no='{$trade_no}' limit 1");
        } elseif(!empty($_GET['out_trade_no'])){
            $out_trade_no = daddslashes($_GET['out_trade_no']);
            $row = $DB->getRow("SELECT * FROM pre_order WHERE uid='{$pid}' and out_trade_no='{$out_trade_no}' limit 1");
        } else {
            exit(json_encode(['code' => -4, 'msg' => '订单号不能为空']));
        }
    }
    if($row){
        $type = $DB->getColumn("SELECT name FROM pre_type WHERE id='{$row['type']}' LIMIT 1");
        $result = array(
            "code" => 1,
            "msg" => "succ",
            "trade_no" => $row['trade_no'],
            "out_trade_no" => $row['out_trade_no'],
            "api_trade_no" => $row['api_trade_no'],
            "type" => $type,
            "pid" => $row['uid'],
            "addtime" => $row['addtime'],
            "endtime" => $row['endtime'],
            "name" => $row['name'],
            "money" => $row['money'],
            "param" => $row['param'],
            "buyer" => $row['buyer'],
            "status" => $row['status'],
            "payurl" => $row['payurl']
        );
    } else {
        $result = array("code" => -1, "msg" => "订单号不存在");
   

​
