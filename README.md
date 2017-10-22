在搜索框中输入关键字,希望实现的效果:

1. 树形图隐藏所有不匹配的节点
2. 节点名称中匹配部分高亮
效果图如下:


完整代码和详细注释如下:
```
<!-- fuzzysearch.html -->
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<link rel="stylesheet" type="text/css"
	href="css/metro.css" />
<style type="text/css">
ul.ztree {
	margin-top: 10px;
	border: 1px solid #617775;
	background: #f0f6e4;
	width: 600px;
	height: 450px;
	overflow-y: scroll;
	overflow-x: auto;
}
div.zTreeDemoBackground {
	width: 600px;
	height: 450px;
	text-align: left;
}

span.search_highlight {
	color: whitesmoke;
	background-color: darkred; 	
}
</style>
<!-- 定义了一些模拟数据的js文件dataset.js -->
<script type="text/javascript" src="js/dataset.js"></script>
<!-- 引入jquery  -->
<script type="text/javascript" src="js/jquery-2.1.1.min.js"></script>
<!-- 引入ztree插件 -->
<script type="text/javascript" src="js/jquery.ztree.all-3.5.min.js"></script>
<!-- 引入ztree插件扩展功能 -->
<script type="text/javascript" src="js/jquery.ztree.exhide-3.5.js"></script>
</head>
<body>
	<div style="width: 600px">
		<input id="keyword" type="search" placeHolder="搜索关键字"/>
	</div>
	<div class="content_wrap">
		<div class="zTreeDemoBackground left">
			<ul id="book" class="ztree"></ul>
		</div>
	</div>
	<br />
	<div><small>*参照亚马逊中文网图书板块的结构创建了模拟数据</small></div>
	<script type="text/javascript">
			
	//此高亮用于整个节点
	/* function setHighlight(treeId, treeNode) {
		return (treeNode.highlight) ? {"font-weight":"bold", "background-color": "#ddd"} : {"font-weight":"normal", "background-color": "transparent"};
	} */
	
	//ztree配置
	var setting = {
			check: {
				enable: true//checkbox
			},
			view: {
				//fontCss: setHighlight, // 高亮一定要设置,setHighlight是自定义方法
				nameIsHTML: true, //允许name支持html				
				selectedMulti: false
			},
			edit: {
				enable: false,
				editNameSelectAll: false
			},
			data: {
				simpleData: {
					enable: true
				}
			}
	};
	
	$(document).ready(function(){
		//loadDataFromServer();//从服务器读取数据并初始化树形图,与loadDataFromLocal()方法互换  
		loadDataFromLocal();//从本地dataset.js文件读取模拟数据并初始化树形图
		fuzzySearch(); //初始化模糊搜索方法
	});

	/**
	 * 从本地js/dataset.js文件读取模拟数据并初始化树形图
	 */
	function loadDataFromLocal(){
		//此处的树节点数据mockData是在本地js/dataSet.js中预先定义的模拟数据
		initThisZtree(mockData);//传入数据,初始化ztree
	}
	
	/**
	 * 通过ajax方法从服务器获取数据并初始化树形图
	 */
	function loadDataFromServer(){
		 $.ajax({                 
			type: "POST",                 
			dataType: "json",                 
			url: 'posturl',  //服务请求地址 
			success: function(data) {   
				initThisZtree(data);//传入数据,初始化ztree
			}     
		});   
	}
	
	/**
	 * 初始化ztree
	 * 
	 * @param {Object} data
	 */
	var treeObj; //zTree对象
	function initThisZtree(data){
		//初始化ztree三个参数分别是(jQuery对象,ztree设置,树节点数据)
		$.fn.zTree.init($("#book"), setting, data); 
		treeObj = $.fn.zTree.getZTreeObj("book"); //根据id获取ztree对象
		treeObj.expandAll(true);
	}
	
	/**
	 * 模糊搜索方法
	 */
	function fuzzySearch(){
		if(!treeObj){
			alter("获取树对象失败");
		}
		
		var hiddenNodes = []; //用于存储被隐藏的节点
		var hiddenNodes2 = []; //用于存储第二次判断后被隐藏的节点
		// 过滤ztree显示数据
		function ztreeFilter() {
			var zTreeObj = $.fn.zTree.getZTreeObj("book");//获取树对象
			// 显示上次搜索后被隐藏的节点
			zTreeObj.showNodes(hiddenNodes);
			var _keywords = $("#keyword").val().toLowerCase(); //获取搜索关键字,直接处理为小写
			var children;//子节点集合
			
			//查看传入的节点集合是否都为隐藏状态,返回boolean
			function isAllChildrenHidden(nodes) {
				isAllHidden = true;
				for (var i = 0; i < nodes.length; i++) {
					if (!nodes[i].isHidden) { //只要有一个节点显示,即返回false
						isAllHidden = false;
						break;
					}
				}
				return isAllHidden;
			}
			
			//如果父节点的子节点都已被隐藏,且父节点不包含关键字,则隐藏当前父节点
			function hideParent(parentNode) {
				if (parentNode && parentNode.name
						&& parentNode.name.toLowerCase().indexOf(_keywords) == -1) {
					children = parentNode.children;
					var isAllHidden = isAllChildrenHidden(children);
					if (isAllHidden) {
						hiddenNodes2.push(parentNode);//将二次判断隐藏的节点存入列表
						zTreeObj.hideNode(parentNode); 
					}
				}
			}
			
			var parent; //父节点
			var preNode; //前一节点
			//为关键字加高亮在原始名称上添加了css样式,得到一个新名称newname
			var newname;
			var metaChar = '[\\[\\]\\\\\^\\$\\.\\|\\?\\*\\+\\(\\)]'; //js正则表达式元字符集
			var rexMeta = new RegExp(metaChar, 'gi');//匹配元字符的正则表达式
			
			// 查找不符合条件的叶子节点
			function filterFunc(node) {
				if(node && node.oldname && node.oldname.length>0){
					node.name = node.oldname; //如果存在原始名称则恢复原始名称
				}
				//node.highlight = false; //取消高亮
				zTreeObj.updateNode(node); //更新节点让之前对节点所做的修改生效
				if (!_keywords || _keywords.length == 0) { //如果关键字为空,返回false
					return false;
				}
				
				preNode = node.getPreNode(); //获取前一个节点
				
				//对前一个未隐藏的父节点进行判断
				if (preNode && preNode.isParent && !preNode.isHidden) {
					hideParent(preNode); //隐藏前一个未隐藏的父节点
				}
				
				if (node.name && node.name.toLowerCase().indexOf(_keywords)!=-1) {
					var newKeywords = _keywords; //创建一个新变量,不对_keywords进行修改
					var _keywordsMeta = newKeywords.match(rexMeta); //检查关键字中是否有元字符
					if(_keywordsMeta){ //如果存在元字符
						$.each(_keywordsMeta,function(i,o){ 
							newKeywords = newKeywords.replace(o,"\\"+o); //对每个元字符进行转义处理
						});
					}
					
					//对原文中关键字加上高亮样式后得到一个新名称newname
					node.oldname = node.name; //缓存原有名称用于恢复
					//创建正则表达式,全局且不分大小写
					var rex = new RegExp(newKeywords, 'gi');//'g'代表全局替换,'i'代表不区分大小写
					var toHighlight = node.name.match(rex)[0]; //获取节点名称中匹配关键字的原始部分用于高亮
					newname = node.name.replace(rex,"<span class='search_highlight'>"+toHighlight+"</span>");
					node.name = newname; //将新名称赋给节点
					//================================================//
					//node.highlight用于高亮整个节点
					//配合setHighlight方法和setting中view属性的fontCss
					//因为有了关键字高亮处理,所以不再进行相关设置
					//node.highlight = true; 
					//================================================//
					zTreeObj.updateNode(node); //update让更名和高亮生效
					return false; //带有关键字的节点不隐藏
				}
				
				if (node.isParent) { //当前节点是父节点不隐藏,等待下一个兄弟节点对其进行隐藏
					return false;
				}
				
				if (node.isLastNode) { //只对最后一个节点操作
					//必须在isLastNode判断之后才能隐藏,因为隐藏后isLastNode均为false
					zTreeObj.hideNode(node); //隐藏当前节点
					parent = node.getParentNode(); // 获取当前节点的父节点
					hideParent(parent); //最后一个节点时才判断父节点是否应该隐藏
				} else {
					hiddenNodes2.push(node); //将二次判断后隐藏的节点存入列表
					zTreeObj.hideNode(node); // 隐藏不符合要求的子节点
				}
				return true;
			}

			// 获取不符合条件的叶子结点
			hiddenNodes = zTreeObj.getNodesByFilter(filterFunc);
			//将二次判断时隐藏的节点加入 hiddenNodes
			hiddenNodes = hiddenNodes.concat(hiddenNodes2); 
		}
		//监听关键字input输入框文字变化事件
		$('#keyword').bind('input propertychange', function() {
			searchNodeLazy(); //调用延时处理
		});

		var timeoutId = null;
		// 有输入后定时执行一次，如果上次的输入还没有被执行，那么就取消上一次的执行
		function searchNodeLazy(value) {
			if (timeoutId) { //如果不为空,结束任务
				clearTimeout(timeoutId);
			}
			timeoutId = setTimeout(function() {
				ztreeFilter();	//延时执行筛选方法
			}, 500);
		}
	}
	</script>
</body>
</html>
```
参考文章:
1. [ztree根据关键字模糊搜索](http://blog.csdn.net/danielpei1222/article/details/52078013)
2. [我也来实现ztree模糊搜索功能](https://www.juwends.com/tech/javascript-tech/ztree_fuzzy_search.html?utm_source=tuicool&utm_medium=referral)
