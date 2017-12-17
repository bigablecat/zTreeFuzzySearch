---
layout: post
title: zTree实现树形图模糊搜索
category: programming
---
### 0. 项目地址:  
[https://github.com/bigablecat/zTreeFuzzySearch](https://github.com/bigablecat/zTreeFuzzySearch)  

### 1. 在搜索框中输入关键字,希望实现的效果:  

a). 树形图隐藏所有不匹配的节点  

b). 节点名称中匹配部分高亮  

### 2. 慢速演示:
![ztree_demo](https://raw.githubusercontent.com/bigable/imgs/master/ztree_demo.gif)  

### 3. 完整代码和详细注释如下:  

a). html部分  

```html
<!-- fuzzysearch.html -->
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<link rel="stylesheet" type="text/css" href="css/metro.css" />
<style type="text/css">
ul.ztree {
	margin-top: 10px;
	border: 1px solid #617775;
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
<script type="text/javascript" src="js/jquery-1.4.4.min.js"></script>
<!-- 引入ztree插件 -->
<script type="text/javascript" src="js/jquery.ztree.all.min.js"></script>
<!-- 引入ztree插件扩展功能 -->
<script type="text/javascript" src="js/jquery.ztree.exhide.min.js"></script>
<!-- 引入自定义方法 -->
<script type="text/javascript" src="js/fuzzysearch.js"></script>
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
				// 使用ztree自定义高亮时，一定要设置fontCss,setHighlight是自定义高亮方法
				//fontCss: setHighlight,
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
		//从服务器读取数据并初始化树形图
		//loadDataFromServer(urlStr);  
		//本例直接从模拟数据dataset.js读取
		loadDataFromLocal();//从本地dataset.js文件读取模拟数据并初始化树形图
	});

	/**
	 * 通过ajax方法从服务器获取数据并初始化树形图
	 */
	function loadDataFromServer(urlStr){
		 $.ajax({                 
			type: "get",                 
			dataType: "json",
			url: urlStr,  //服务请求地址 
			success: function(data) {
				initThisZtree(data);//传入数据,初始化ztree
				fuzzySearch('book','#keyword',null,true); //初始化模糊搜索方法
			}     
		});   
	}
	
	/**
	 * 从本地js/dataset.js文件读取模拟数据并初始化树形图
	 */
	function loadDataFromLocal(){
		//此处的树节点数据mockData是在本地js/dataSet.js中预先定义的模拟数据
		initThisZtree(mockData);//传入数据,初始化ztree
		// zTreeId ztree对象的id,不需要#
 		// searchField 输入框选择器
        // isHighLight 是否高亮,默认高亮,传入false禁用
        // isExpand 是否展开,默认合拢,传入true展开
		fuzzySearch('book','#keyword',null,true); //初始化模糊搜索方法
	}
	
	/**
	 * 初始化ztree
	 * 
	 * @param {Object} data
	 */
	function initThisZtree(data){
		//初始化ztree三个参数分别是(jQuery对象,ztree设置,树节点数据)
		var treeObj = $.fn.zTree.init($("#book"), setting, data); 
		treeObj.expandAll(true);
	}
	</script>
</body>
</html>
```

a). js部分  

```html
<!-- fuzzysearch.js -->
/**
 * 
 * @param zTreeId ztree对象的id,不需要#
 * @param searchField 输入框选择器
 * @param isHighLight 是否高亮,默认高亮,传入false禁用
 * @param isExpand 是否展开,默认合拢,传入true展开
 * @returns
 */	
 function fuzzySearch(zTreeId, searchField, isHighLight, isExpand){
	var zTreeObj = $.fn.zTree.getZTreeObj(zTreeId);//获取树对象
	if(!zTreeObj){
		alter("获取树对象失败");
	}
	var nameKey = zTreeObj.setting.data.key.name; //获取name属性的key
	isHighLight = isHighLight===false?false:true;//除直接输入false的情况外,都默认为高亮
	isExpand = isExpand?true:false;
	zTreeObj.setting.view.nameIsHTML = isHighLight;//允许在节点名称中使用html,用于处理高亮
	
	var metaChar = '[\\[\\]\\\\\^\\$\\.\\|\\?\\*\\+\\(\\)]'; //js正则表达式元字符集
	var rexMeta = new RegExp(metaChar, 'gi');//匹配元字符的正则表达式
	
	// 过滤ztree显示数据
	function ztreeFilter(zTreeObj,_keywords,callBackFunc) {
		if(_keywords){
			_keywords.toLowerCase(); //获取搜索关键字,直接处理为小写	
		}else{
			_keywords =''; //空字符串
		}
		
		// 查找不符合条件的叶子节点
		function filterFunc(node) {
			if(node && node.oldname && node.oldname.length>0){
				node[nameKey] = node.oldname; //如果存在原始名称则恢复原始名称
			}
			//node.highlight = false; //取消高亮
			zTreeObj.updateNode(node); //更新节点让之前对节点所做的修改生效
			if (_keywords.length == 0) { 
				//如果关键字为空,返回false,表示每个节点都显示
				zTreeObj.showNode(node);
				zTreeObj.expandNode(node,isExpand); //关键字为空时是否展开节点
				return true;
			}
			//节点名称和关键字都用toLowerCase()做小写处理
			if (node[nameKey] && node[nameKey].toLowerCase().indexOf(_keywords.toLowerCase())!=-1) {
				if(isHighLight){ //如果高亮，对文字进行高亮处理
					//创建一个新变量newKeywords,不影响_keywords在下一个节点使用
					//对_keywords中的元字符进行处理,否则无法在replace中使用RegExp
					var newKeywords = _keywords.replace(rexMeta,function(matchStr){
						return "\\" + matchStr; //对元字符做转义处理
					});
					//为处理过元字符的_keywords创建正则表达式,全局且不分大小写
					var rexGlobal = new RegExp(newKeywords, 'gi');//'g'代表全局匹配,'i'代表不区分大小写
					//获取节点名称中匹配关键字的原始部分用于高亮
					var originalText = node[nameKey].match(rexGlobal)[0];
					var highLightText = 
						'<span style="color: whitesmoke;background-color: darkred;">'
						+ originalText
						+'</span>';
					node.oldname = node[nameKey]; //缓存原有名称用于恢复
					//无法直接使用replace(/substr/g,replacement)方法,所以使用RegExp
					node[nameKey] = node.oldname.replace(rexGlobal, highLightText);//将关键字替换为高亮文本
					//================================================//
					//node.highlight用于高亮整个节点
					//配合setHighlight方法和setting中view属性的fontCss
					//因为有了关键字高亮处理,所以不再进行相关设置
					//node.highlight = true; 
					//================================================//
					zTreeObj.updateNode(node); //update让更名和高亮生效
				}
				zTreeObj.showNode(node);//显示符合条件的节点
				return true; //带有关键字的节点不隐藏
			}
			
			zTreeObj.hideNode(node); // 隐藏不符合要求的节点
			return false;
		}
		var nodesShow = zTreeObj.getNodesByFilter(filterFunc); //获取匹配关键字的节点
		processShowNodes(nodesShow,_keywords);//对获取的节点进行处理
	}
	
	/**
	 * 对符合条件的节点做二次处理
	 */
	function processShowNodes(nodesShow,_keywords){
		if(nodesShow && nodesShow.length>0){
			//关键字不为空时对关键字节点的祖先节点进行二次处理
			if(_keywords.length>0){ 
				$.each(nodesShow,function(n,obj){
					var pathOfOne = obj.getPath();//向上追溯,获取节点的所有父节点(包括自己)
					if(pathOfOne && pathOfOne.length>0){ //对path中的每个节点进行操作
						//i<pathOfOne.length-1,对节点自己不再操作
						for(var i=0;i<pathOfOne.length-1;i++){
							zTreeObj.showNode(pathOfOne[i]); //显示节点
							zTreeObj.expandNode(pathOfOne[i],true); //展开节点
						}
					}
				});				
			}else{ //显示所有节点时展开根节点
				var rootNodes = zTreeObj.getNodesByParam('level','0');//获得所有根节点
				$.each(rootNodes,function(n,obj){
					zTreeObj.expandNode(obj,true); //展开所有根节点
				});
			}
		}
	}
	
	//监听关键字input输入框文字变化事件
	$(searchField).bind('input propertychange', function() {
		var _keywords = $(this).val();
		searchNodeLazy(_keywords); //调用延时处理
	});

	var timeoutId = null;
	// 有输入后定时执行一次，如果上次的输入还没有被执行，那么就取消上一次的执行
	function searchNodeLazy(_keywords) {
		if (timeoutId) { //如果不为空,结束任务
			clearTimeout(timeoutId);
		}
		timeoutId = setTimeout(function() {
			ztreeFilter(zTreeObj,_keywords);	//延时执行筛选方法
			$(searchField).focus();//输入框重新获取焦点
		}, 500);
	}
}
```

### 4.参考文章:

a). [ztree根据关键字模糊搜索](http://blog.csdn.net/danielpei1222/article/details/52078013)
b). [我也来实现ztree模糊搜索功能](https://www.juwends.com/tech/javascript-tech/ztree_fuzzy_search.html?utm_source=tuicool&utm_medium=referral)
