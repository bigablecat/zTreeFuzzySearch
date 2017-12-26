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

b). js部分  

```javascript  
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
		if(!_keywords){
			_keywords =''; //如果为空，赋值空字符串
		}
		
		// 查找符合条件的叶子节点
		function filterFunc(node) {
			if(node && node.oldname && node.oldname.length>0){
				node[nameKey] = node.oldname; //如果存在原始名称则恢复原始名称
			}
			//node.highlight = false; //取消高亮
			zTreeObj.updateNode(node); //更新节点让之前对节点所做的修改生效
			if (_keywords.length == 0) { 
				//如果关键字为空,返回true,表示每个节点都显示
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
						//对元字符做转义处理
						return '\\' + matchStr;
						
					});
					node.oldname = node[nameKey]; //缓存原有名称用于恢复
					//为处理过元字符的_keywords创建正则表达式,全局且不分大小写
					var rexGlobal = new RegExp(newKeywords, 'gi');//'g'代表全局匹配,'i'代表不区分大小写
					//无法直接使用replace(/substr/g,replacement)方法,所以使用RegExp
					node[nameKey] = node.oldname.replace(rexGlobal, function(originalText){
						//将所有匹配的子串加上高亮效果
						var highLightText =
							'<span style="color: whitesmoke;background-color: darkred;">'
							+ originalText
							+'</span>';
						return 	highLightText;					
					});
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
			return false; //不符合返回false
		}
		var nodesShow = zTreeObj.getNodesByFilter(filterFunc); //获取匹配关键字的节点
		processShowNodes(nodesShow, _keywords);//对获取的节点进行二次处理
	}
	
	/**
	 * 对符合条件的节点做二次处理
	 */
	function processShowNodes(nodesShow,_keywords){
		if(nodesShow && nodesShow.length>0){
			//关键字不为空时对关键字节点的祖先节点进行二次处理
			if(_keywords.length>0){ 
				$.each(nodesShow, function(n,obj){
					var pathOfOne = obj.getPath();//向上追溯,获取节点的所有祖先节点(包括自己)
					if(pathOfOne && pathOfOne.length>0){ //对path中的每个节点进行操作
						// i < pathOfOne.length-1, 对节点本身不再操作
						for(var i=0;i<pathOfOne.length-1;i++){
							zTreeObj.showNode(pathOfOne[i]); //显示节点
							zTreeObj.expandNode(pathOfOne[i],true); //展开节点
						}
					}
				});				
			}else{ //关键字为空则显示所有节点, 此时展开根节点
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

c). 模拟数据部分  

```json  
/*dataset.js*/
var mockData = [{
	"name": "图书",
	"pId": "",
	"id": "0"
}, {
	"name": "其他1",
	"pId": "",
	"id": "1"
},{
	"name": "其他2",
	"pId": "",
	"id": "2"
},{
	"name": "其他3",
	"pId": "",
	"id": "3"
},
{
	"name": "测试js正则元字符:[]\\^$.|?*+()",
	"pId": "0",
	"id": "06"
},{
	"name": "测试其他字符:{}<>'\"~`!@#%&-;:/,",
	"pId": "0",
	"id": "07"
},{
	"name": "测试大小写同节点: 大写ABDEFGHINQRT,小写abdefghinqrt",
	"pId": "0",
	"id": "08"
},{
	"name": "测试大写ABDEFGHINQRT",
	"pId": "0",
	"id": "09"
},{
	"name": "测试小写abdefghinqrt",
	"pId": "0",
	"id": "10"
},{
	"name": "文学",
	"pId": "0",
	"id": "00"
}, {
	"name": "历史",
	"pId": "0",
	"id": "01"
}, {
	"name": "法律",
	"pId": "0",
	"id": "02"
}, {
	"name": "辞典与工具书",
	"pId": "0",
	"id": "03"
}, {
	"name": "科学与自然",
	"pId": "0",
	"id": "04"
},{
	"name": "自然科学总论",
	"pId": "04",
	"id": "040"
},{
	"name": "数学",
	"pId": "04",
	"id": "041"
},{
	"name": "物理",
	"pId": "04",
	"id": "042"
},{
	"name": "天文学 地球科学",
	"pId": "04",
	"id": "043"
}, {
	"name": "计算机与互联网",
	"pId": "0",
	"id": "05"
}, {
	"name": "编程与开发",
	"pId": "05",
	"id": "050"
}, {
	"name": "编程语言与工具",
	"pId": "050",
	"id": "0500"
}, {
	"name": "C & C++",
	"pId": "0500",
	"id": "05000"
},{
	"name": "JAVA",
	"pId": "0500",
	"id": "05001"
},{
	"name": "Python",
	"pId": "0500",
	"id": "05002"
},{
	"name": "Python编程 从入门到实践",
	"pId": "05002",
	"id": "050020"
},{
	"name": "Python编程快速上手 让繁琐工作自动化",
	"pId": "05002",
	"id": "050021"
}, {
	"name": "算法与数据结构",
	"pId": "050",
	"id": "0501"
}, {
	"name": "编译原理和编译器",
	"pId": "050",
	"id": "0502"
}, {
	"name": "操作系统",
	"pId": "05",
	"id": "051"
}, {
	"name": "数据库",
	"pId": "05",
	"id": "052"
}, {
	"name": "计算机科学理论",
	"pId": "05",
	"id": "053"
}, {
	"name": "云计算与大数据",
	"pId": "05",
	"id": "054"
}]
```  
### 4.参考资料:  

a). [ztree根据关键字模糊搜索](http://blog.csdn.net/danielpei1222/article/details/52078013)
b). [我也来实现ztree模糊搜索功能](https://www.juwends.com/tech/javascript-tech/ztree_fuzzy_search.html?utm_source=tuicool&utm_medium=referral)
