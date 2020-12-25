---
layout:     post
title:      "Excel 处理"
subtitle:   " \"import and  export\""
date:       2020-12-22 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
---

> 千里之行，始于足下

### 序言

工作中，总是会有各种基础数据，以报表数据导出的需求，通常是以Excel导出导入比较普遍。Excel导出，通常分为查询列表的列表数据导出和用户固定模板汇总导出。所以记录处理excel的API,为了更好的提高处理效率和方便使用。

### 1、Easy POI(首选)

固定模板导出，嵌套等复杂模板导出的福音。功能性强大。提高开发速度，缩短代码量。excel导入导出的第一选择，有待研究和实战。

项目地址：

<https://gitee.com/lemur/easypoi>

使用教程：

<https://opensource.afterturn.cn/doc/easypoi.html#4>

<http://easypoi.mydoc.io/>

[测试项目](http://git.oschina.net/lemur/easypoi-test): 

<http://git.oschina.net/lemur/easypoi-test>



### 2、Easyexcel（次之）

EasyExcel是一个基于Java的简单、省内存的读写Excel的开源项目，对简单读写操作比较实用。

项目地址：<https://github.com/alibaba/easyexcel>

使用文档：<https://alibaba-easyexcel.github.io/>



### 3、Apache POI

工作中最常使用使用的api，方便快捷，但是不够灵活，特别是处理固定模板导出的时候，基本要靠自己编码画模板，所以会出现模板样式不准确的问题。所以在工作需要注意以下几个问题。

* 导出文件名，中文名称乱码的问题。-》   设置编码格式
* 导入excel文件时，不支持xlsx,只支持xls 。-》 api的版本过低问题
* 画模板时，准确设置列宽问题 。 -》[java用POI设置Excel的列宽](https://blog.csdn.net/duqian42707/article/details/51491312)
* 导入excel时，多行数据做校验。校验出错时，数据回滚问题。 -》1、可以抛异常时，手动设置回滚和提交。2、遍历所有行，将报错信息全部记录在一个错误字符串，正确的数据，暂存入List集合，最后校验是否存在错误，没有就批量更新。（推荐）
* 导出列表通用性问题。 -》将所有的导出列表转变为一个JavaBean，可以通过反射获取属性值，编写一个通用的方法导出。
* java - poi - excel导入读取行数不正确 | java - poi - excel导入无法获取正确行数 | getLastRowNum()行数不正确 | getPhysicalNumberOfRows()行数不正确 -》重新下载模板，将数据重新拷贝过去，重新导入，由于格式已经被用户打乱。[解决的方法](https://www.jianshu.com/p/b36c9230089f)
  

#### 3.1 excel模板导出样例

实战中，效果如图，主要难点，合并单元格和样式调整，样式这块不如easy-poi 方便，easy-poi直接使用客户模板，不用画。

![excel export](\img\java\excel\exportExport.png)

exportRecipes.java

```java
    @SuppressWarnings("rawtypes")
	@ApiOperation("导出食谱明细")
    @PostMapping("/exportRecipes")
    @ResponseBody
    public Result exportRecipes(@RequestParam("ids") String allIds, HttpServletResponse response, HttpServletRequest request) throws Exception {
    	Result result = new Result();
        //List<Map<String,Object>> list = new ArrayList<Map<String,Object>>();
        List<RecipeseExportVo> list = new ArrayList<RecipeseExportVo>();
    try {
        Recipes recipes = recipesMapper.selectByPrimaryKey(allIds);
        String week = recipes.getWeek();
        String canteenName = recipes.getCanteenName();
        String version =  recipes.getVersion();
        String canteenId = recipes.getCanteenId();
        //查询所有的窗口和餐次
        List<String> canlist = new ArrayList<String>();
        List<Map<String,String>> cantenCountList = new ArrayList<>();
        List<Canteen> canteenList = canteenMapper.getWindowByCanteen(canteenId);
        BaseParameterExample baseParameterExample = new BaseParameterExample();
        BaseParameterExample.Criteria criteria = baseParameterExample.createCriteria();
        criteria.andPtypeEqualTo("meal_times");
        baseParameterExample.setOrderByClause(" pseq asc");
        List<BaseParameter> parameterList = baseParameterMapper.selectByExample(baseParameterExample);
        for(Canteen canteen:canteenList){
            for(BaseParameter baseParameter: parameterList){
            	canlist.add(canteen.getName()+";"+baseParameter.getPvalue());
                RecipeseExportVo recipeseExportVo = new RecipeseExportVo();
                recipeseExportVo.setRecipesWindowName(canteen.getName());
                recipeseExportVo.setRecipesCountName(baseParameter.getPvalue());
                list.add(recipeseExportVo);
            }
        }
        List<Map<String,Object>> recipesList = recipesMapper.queryDistinctById(allIds);
        if(recipesList !=null){
            recipesList = CommonUtils.sourceTofomater(recipesList);
        }
        for(Map<String,Object> map : recipesList) {
            Map<String,Object> resultMap = new HashMap<String,Object>();
            String recipesWindow = CommonUtils.getMapStr(map,"recipesWindow","");
            String recipesCount = CommonUtils.getMapStr(map,"recipesCount","");
            String recipesCountName = baseParameterMapper.getValueBykey("meal_times",recipesCount);
            String recipesWindowName = CommonUtils.getMapStr(map,"recipesWindowName","");
            Integer listSize = canlist.indexOf(recipesWindowName+";"+recipesCountName);

            Integer size = 0; //获取某天最大的食品
            for(int i=1;i<=7;i++){
                List<Map<String,Object>> food = new ArrayList<Map<String,Object>>();
                List<Map<String,Object>> foodList = recipesDetailMapper.queryByRecipes(recipesWindow,recipesCount,String.valueOf(i),allIds);
                Integer foodSize =0; //查询食品的数量
                if(foodList !=null && foodList.size()>0 ) {
                    for (Map<String, Object> fooMap : foodList) {
                        if (fooMap != null) {
                            fooMap = CommonUtils.sourceTofomater(fooMap);
                            food.add(fooMap);
                            if(food.size()>0){
                                resultMap.put(String.valueOf(i),food);
                            }
                            foodSize++;
                        }
                    }
                }
                size=(foodSize < size?size: foodSize);
            }
            //当天最大
            for(int i=0;i<size;i++){
                RecipeseExportVo vo = new RecipeseExportVo();
                vo.setRecipesWindowName(recipesWindowName);
                vo.setRecipesCountName(recipesCountName);
                //遍历每一天，取相应的记录数
                List<Map<String,Object>> monFoodList = (List<Map<String, Object>>) resultMap.get("1");
                if(monFoodList != null && monFoodList.size()>i){
                    Map<String,Object> monFood = monFoodList.get(i);
                    vo.setMonFood(monFood.get("foodName").toString());
                }
                List<Map<String,Object>> tuesFoodList = (List<Map<String, Object>>) resultMap.get("2");
                if(tuesFoodList != null && tuesFoodList.size()>i){
                    Map<String,Object> tuesFood = tuesFoodList.get(i);
                    vo.setTuesFood(tuesFood.get("foodName").toString());
                }
                List<Map<String,Object>> wedFoodList = (List<Map<String, Object>>) resultMap.get("3");
                if(wedFoodList != null && wedFoodList.size()>i){
                    Map<String,Object> wedFood = wedFoodList.get(i);
                    vo.setWedFood(wedFood.get("foodName").toString());
                }
                List<Map<String,Object>> thurFoodList = (List<Map<String, Object>>) resultMap.get("4");
                if(thurFoodList != null && thurFoodList.size()>i){
                    Map<String,Object> thurFood = thurFoodList.get(i);
                    vo.setThurFood(thurFood.get("foodName").toString());
                }
                List<Map<String,Object>> friFoodList = (List<Map<String, Object>>) resultMap.get("5");
                if(friFoodList != null && friFoodList.size()>i){
                    Map<String,Object> friFood = friFoodList.get(i);
                    vo.setFriFood(friFood.get("foodName").toString());
                }
                List<Map<String,Object>> satFoodList = (List<Map<String, Object>>) resultMap.get("6");
                if(satFoodList != null && satFoodList.size()>i){
                    Map<String,Object> satFood = satFoodList.get(i);
                    vo.setSatFood(satFood.get("foodName").toString());
                }
                List<Map<String,Object>> sunFoodList = (List<Map<String, Object>>) resultMap.get("7");
                if(sunFoodList != null && sunFoodList.size()>i){
                    Map<String,Object> sunFood = sunFoodList.get(i);
                    vo.setSunFood(sunFood.get("foodName").toString());
                }
                //list.set(i+listSize,vo);
                list.add(i+listSize,vo);
                canlist.add(i+listSize,recipesWindowName+";"+recipesCountName);
            }
            	list.remove(size+listSize);
            	canlist.remove(size+listSize);
        }
        Integer size = list.size();
    	Map<String, Object> map = new HashMap<String,Object>();
    	String sheetName = "菜品明细";
		//表头创建两行，用于合并单元格 start
		List<String> fields1 = new ArrayList<String>();
		fields1.add("窗口");
		fields1.add("餐次");
		fields1.add("星期一");
		fields1.add("星期二");
		fields1.add("星期三");
		fields1.add("星期四");
		fields1.add("星期五");
		fields1.add("星期六");
		fields1.add("星期日");
		//表头创建两行，用于合并单元格 end
		HSSFWorkbook workbook = new HSSFWorkbook();// 产生工作薄对象		
		//字体样式
		HSSFCellStyle fontStyle = workbook.createCellStyle();
		HSSFFont font = workbook.createFont();
		font.setFontHeightInPoints((short) 10);
		font.setFontName("宋体");
		fontStyle.setFont(font);

		//单元格样式
		HSSFCellStyle cellStyle = workbook.createCellStyle();
		HSSFFont cellFont = (HSSFFont) workbook.createFont();	//创建字体样式
		cellFont.setFontName("宋体");	//设置字体类型
		cellFont.setFontHeightInPoints((short) 10);	//设置字体大小
		cellStyle.setFont(cellFont);	//为标题样式设置字体样式
		cellStyle.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);	//设置垂直居中
		cellStyle.setFillForegroundColor(HSSFColor.LIGHT_YELLOW.index);
		cellStyle.setBorderBottom(HSSFCellStyle.BORDER_THIN); // 下边框
		cellStyle.setBorderLeft(HSSFCellStyle.BORDER_THIN);// 左边框
		cellStyle.setBorderTop(HSSFCellStyle.BORDER_THIN);// 上边框
		cellStyle.setBorderRight(HSSFCellStyle.BORDER_THIN);// 右边框
		cellStyle.setAlignment(HSSFCellStyle.ALIGN_LEFT);
		cellStyle.setWrapText(true);
        
        HSSFCellStyle cellStyleFirst = workbook.createCellStyle();
        cellStyleFirst.setFont(cellFont);
        cellStyleFirst.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);	//设置垂直居中
		cellStyleFirst.setAlignment(HSSFCellStyle.ALIGN_CENTER);	//设置水平居中
		cellStyleFirst.setFillForegroundColor(HSSFColor.LIGHT_YELLOW.index);
		cellStyleFirst.setBorderBottom(HSSFCellStyle.BORDER_THIN); // 下边框
		cellStyleFirst.setBorderLeft(HSSFCellStyle.BORDER_THIN);// 左边框
		cellStyleFirst.setBorderTop(HSSFCellStyle.BORDER_THIN);// 上边框
		cellStyleFirst.setBorderRight(HSSFCellStyle.BORDER_THIN);// 右边框
        cellStyleFirst.setFillForegroundColor(HSSFColor.GOLD.index);
        cellStyleFirst.setFillPattern(HSSFCellStyle.SOLID_FOREGROUND);
        cellStyleFirst.setFillBackgroundColor(HSSFColor.GOLD.index);
        cellStyleFirst.setWrapText(true);
        
        HSSFCellStyle cellStyleSecond = workbook.createCellStyle();
        cellStyleSecond.setFont(cellFont);
        cellStyleSecond.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);	//设置垂直居中
        cellStyleSecond.setAlignment(HSSFCellStyle.ALIGN_CENTER);	//设置水平居中
        cellStyleSecond.setFillForegroundColor(HSSFColor.LIGHT_YELLOW.index);
        cellStyleSecond.setBorderBottom(HSSFCellStyle.BORDER_THIN); // 下边框
        cellStyleSecond.setBorderLeft(HSSFCellStyle.BORDER_THIN);// 左边框
        cellStyleSecond.setBorderTop(HSSFCellStyle.BORDER_THIN);// 上边框
        cellStyleSecond.setBorderRight(HSSFCellStyle.BORDER_THIN);// 右边框
        cellStyleSecond.setWrapText(true);
		
		//第一行样式
		HSSFCellStyle oneLineStyle = (HSSFCellStyle) workbook .createCellStyle();// 创建标题样式
		oneLineStyle.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);	//设置垂直居中
		oneLineStyle.setAlignment(HSSFCellStyle.ALIGN_CENTER);	//设置水平居中
		oneLineStyle.setBorderRight(HSSFCellStyle.BORDER_THIN);// 右边框
		HSSFFont oneLineFont = (HSSFFont) workbook.createFont();	//创建字体样式
		oneLineFont.setFontName("宋体");	//设置字体类型
		oneLineFont.setFontHeightInPoints((short) 10);	//设置字体大小
		oneLineFont.setBoldweight(HSSFFont.BOLDWEIGHT_BOLD); // 字体加粗
		oneLineStyle.setFont(oneLineFont);	//为标题样式设置字体样式
		oneLineStyle.setWrapText(true);
        
        
		
		//标题样式
		HSSFCellStyle titleStyle = (HSSFCellStyle) workbook .createCellStyle();// 创建标题样式
		titleStyle.setVerticalAlignment(HSSFCellStyle.VERTICAL_CENTER);	//设置垂直居中
		titleStyle.setAlignment(HSSFCellStyle.ALIGN_CENTER);	//设置水平居中
        titleStyle.setFillForegroundColor(HSSFColor.GOLD.index);
        titleStyle.setFillPattern(HSSFCellStyle.SOLID_FOREGROUND);
        titleStyle.setFillBackgroundColor(HSSFColor.GOLD.index);
		HSSFFont titleFont = (HSSFFont) workbook.createFont();	//创建字体样式
		titleFont.setFontName("宋体");	//设置字体类型
		titleFont.setFontHeightInPoints((short) 11);	//设置字体大小
		titleStyle.setFont(titleFont);	//为标题样式设置字体样式
		titleStyle.setBorderBottom(HSSFCellStyle.BORDER_THIN); // 下边框
		titleStyle.setBorderLeft(HSSFCellStyle.BORDER_THIN);// 左边框
		titleStyle.setBorderTop(HSSFCellStyle.BORDER_THIN);// 上边框
		titleStyle.setBorderRight(HSSFCellStyle.BORDER_THIN);// 右边框

        HSSFSheet sheet = workbook.createSheet();// 产生工作表对象
		workbook.setSheetName(0, sheetName);

		HSSFRow row;
		HSSFCell cell; // 产生单元格

        //获取合并单元格地址，窗口
        List<String> windowsNameList = new ArrayList<String>();//窗口名称
        List<String> countList = new ArrayList<String>();//餐次名称
        for(RecipeseExportVo vo : list){
            windowsNameList.add(vo.getRecipesWindowName());
            countList.add(vo.getRecipesWindowName()+vo.getRecipesCountName());
        }
        List<String> repeatWindowName = getDuplicateElements(windowsNameList);
        List<String> repeatCount = getDuplicateElements(countList);
        
		//创建第一行 start
		row = sheet.createRow(0);
		for (int i = 0; i <= 8; i++) {
			cell = row.createCell(i); // 创建列
			if (i == 0) {
			    String year = week.substring(0,4);
			    String DateStr = DateUtil.getWeekDays(Integer.valueOf(year),0,Integer.valueOf(week.substring(4,6)));
				cell.setCellValue(week+"周套餐菜谱（"+canteenName+"）　          日期："+DateStr);
			}
			cell.setCellStyle(oneLineStyle);
		}
		row.setHeight((short)620);
		sheet.addMergedRegion(new CellRangeAddress(0, 0, 0,8));
		//创建第一行 end
		
		//创建第6行
		row = sheet.createRow(1);
		row.setHeight((short)500);
		
		for (int i = 0; i < fields1.size(); i++) {
			String field = fields1.get(i);

			cell = row.createCell(i); // 创建列
			cell.setCellType(HSSFCell.CELL_TYPE_STRING); // 设置列中写入内容为String类型
			if(i == 0 || i==1) { //第二列，服务项目名称，稍微大一些
			sheet.setColumnWidth(i,256*8+184);
			}else {
			sheet.setColumnWidth(i,256*15+184);
			}
			cell.setCellStyle(titleStyle);
			cell.setCellValue(field);
		}
		// 写入各条记录,每条记录对应excel表中的一行
		HSSFCellStyle cs = workbook.createCellStyle();
		cs.setAlignment(HSSFCellStyle.ALIGN_CENTER);
		cs.setVerticalAlignment(CellStyle.VERTICAL_CENTER);
		for (int i = 0; i < list.size(); i++) {
            RecipeseExportVo recipeseExportVo = list.get(i);
			row = sheet.createRow(i + 2);
			row.setHeight((short)240);
			
			cell = row.createCell(0); //创建cell
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getRecipesWindowName());
			cell.setCellStyle(cellStyleFirst);

			cell = row.createCell(1);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getRecipesCountName());
			cell.setCellStyle(cellStyleSecond);

			cell = row.createCell(2);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getMonFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(3);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getTuesFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(4);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getWedFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(5);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getThurFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(6);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getFriFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(7);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getSatFood());
			cell.setCellStyle(cellStyle);
			
			cell = row.createCell(8);
			cell.setCellStyle(cs);
			cell.setCellValue(recipeseExportVo.getSunFood());
			cell.setCellStyle(cellStyle);
		}
        //动态合并单元格。根据订单号来
        for (String windowname : repeatWindowName) {
            Integer start = windowsNameList.indexOf(windowname) + 2;
            Integer end = windowsNameList.lastIndexOf(windowname) + 2;
            sheet.addMergedRegion(new CellRangeAddress(start, end, 0, 0));
        }

        for (String canteenCount : repeatCount) {
            Integer start = countList.indexOf(canteenCount) + 2;
            Integer end = countList.lastIndexOf(canteenCount) + 2;
            sheet.addMergedRegion(new CellRangeAddress(start, end, 1, 1));
        }

		row = sheet.createRow(2 + list.size());
		row.setHeight((short)850);
		for (int i = 0; i <= 8; i++) {
			cell = row.createCell(i); // 创建列
			if (i == 0) {
				cell.setCellValue("备注 1、由餐饮公司制定周菜谱，每周五上午12：00前将下周菜谱发到膳食办审定；                                                                                                                                                                                                                                                 2、经过膳食委评议合格的新菜品，在自评议之日起一个月时间内推出，每天至少有1个新菜品推出；                                                                                                              3、由于客观原因需要调整当天菜谱（变更、增加、减少）需在9:00前向膳食办申报，得批准后方可执行，一周不得超过五种。");
				cell.setCellType(HSSFCell.CELL_TYPE_STRING);
				
			}
			cell.setCellStyle(cellStyle);
		}
		sheet.addMergedRegion(new CellRangeAddress(2 + list.size(), 2 + list.size(), 0, 8));
		row = sheet.createRow(3 + list.size());
		row.setHeight((short)360);
		for (int i = 0; i <= 8; i++) {
			cell = row.createCell(i); // 创建列
			if (i == 0) {
				cell.setCellValue("拟定：长春餐饮公司                                                    审核：膳食办");
				cellStyle.setAlignment(HSSFCellStyle.ALIGN_LEFT);
				cell.setCellType(HSSFCell.CELL_TYPE_STRING);
			}
			cell.setCellStyle(cellStyle);
	
		}
		sheet.addMergedRegion(new CellRangeAddress(3 + list.size(), 3 + list.size(), 0, 8));
		String filename = "byfront";
        ExcelUtil.setResponseHeader(response, filename,request);
        OutputStream os = response.getOutputStream();
        workbook.write(os);
        os.flush();
        os.close();
		} catch (Exception e) {
			  LOGGER.error("=====删除异常:{}=====", e);
	            result.setCode(ResultCodes.ERROR);
	            result.setMsg(ResultCodes.ERROR_MSG);
		}
		return null;
    }


	//获取重复的元素。
	public static <E> List<E> getDuplicateElements(List<E> list) {
		return list.stream() // list 对应的 Stream
				.collect(Collectors.toMap(e -> e, e -> 1, (a, b) -> a + b)) // 获得元素出现频率的 Map，键为元素，值为元素出现的次数
				.entrySet().stream() // 所有 entry 对应的 Stream
				.filter(entry -> entry.getValue() > 1) // 过滤出元素出现次数大于 1 的 entry
				.map(entry -> entry.getKey()) // 获得 entry 的键（重复元素）对应的 Stream
				.collect(Collectors.toList()); // 转化为 List
	}
```



