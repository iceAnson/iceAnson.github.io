---
layout: post
title: ChartView 图表控件
description: chartview for android 2015-10-14
category: blog
---

地址：[此处输入链接的描述][https://github.com/iceAnson/ChartView]
##一、介绍

ChartView是一个extends view的自定义图标控件
主要提供了：
    

 - 图表格子颜色宽度自定义
 - 横刻度 竖刻度 自定义范围和颜色，支持int类型和float类型
 - 点线连接支持贝塞尔曲线连接，平滑过渡效果
 - 点线连接支持是否绘制闭合区域，可自定义区域颜色
 - 支持绘制区域颜色，主要用于标记正常范围的场景
 - 支持游标
 - 左右滑动 阻尼效果

##二、使用参数如下
使用方法只需要按如下配置响应参数，然后执行初始化即可；
若需要更新数据，可直接使用ChartViewConfig 设置相应的方法，再ChartView.update即可实现刷新；


                //初始化
                ChartViewConfig config = new ChartViewConfig();config
                //设置列数
                .setCloumn(4)
                //设置行数
                .setRow(10)
                //设置每一个格子的高度
                .setItem_height(80)
                //设置每一个格子的宽度
                .setItem_width(mScreenWidth / 4)
                //设置格子线的颜色
                .setGrid_line_color(R.color.rcharview_grid_line_color)
                //设置横竖刻度线的颜色
                .setGrid_line_kedu_color(R.color.xiyou_white)
                //设置是否显示格子线，PS:即使不显示格子线，以上的参数还是必须设置
                .setIsShowGridLine(false)
                
                        //设置竖向刻度左边距
                .setVerical_kedu_leftmargin(mScreenWidth / 8)
                        //设置竖向刻度单位文案
                .setVerical_unit_text("KG")
                        //设置竖向刻度开始值
                .setVerical_unit_start(0)
                        //设置竖向刻度结束值
                .setVerical_unit_end(15)
                        //设置竖向刻度增量
                .setVerical_unit_incremetal(3)
                        //设置竖向刻度值的类型，支持int和float
                .setVerical_lable_value_type(0)
                        //竖向刻度是否分段
                .setVerical_need_to_fragment(false)
                        //竖向刻度值颜色
                .setVerical_unit_color(R.color.xiyou_white)
                        //竖向刻度 文案 颜色
                .setVerical_unit_lable_color(R.color.xiyou_white)
                        //竖向刻度 文案 颜色
                .setVerical_unit_lable_sub_color(R.color.xiyou_white)
                        //竖向刻度 凸出线是否显示
                .setVerical_kedu_line_show(true)

                        //设置水平刻度
                .setListHorizontalKeduAndValueType(listHorizontal, 0,"1")
                        //设置贝塞尔区域
                .setListPointRegion(listPointRegion)
                        //设置区域颜色
                .setRegion_color(R.color.chart_view_region)
                        //设置点的内容
                .setListPoint(listPoint)
                        //是否平滑过渡，即贝塞尔曲线过度
                .setIsSmoothPoint(true)
                        //是否点线闭合,闭合则需要设置闭合区域颜色
                .setIsFillPointRegion(true)
                        //点线闭合区域颜色
                .setRegion_connect_color(R.color.xiyou_white)

                        //设置点和线的颜色
                .setPath_line_color(R.color.xiyou_white)
                    //保留
                .setPoint_circle_color_interval(R.color.xiyou_white)
                    //保留
                .setPoint_circle_color_outside(R.color.xiyou_white)

                        //游标的单位文本
                .setIndicator_title_unit("kg")
                       //游标颜色
                .setIndicator_color(R.color.xiyou_white)
                       //游标标题颜色
                .setIndicator_title_color(R.color.xiyou_blue)
                       //游标是否跟着曲线一起上下移动
                .setIsIndicatorMoveWithPoint(true)
                       //自定义游标背景
                .setIndicatorBgRes(R.drawable.tree50)
                      //设置游标半径，默认游标是圆形的
                .setIndicator_radius(100)
					 //是否显示游标
				.setIsShowIndicator(true).
                    //默认选中位置
                .setItemSelection(1)
        ;
        chartview.init(config);
        
    
效果图如下：

![](http://7xnby9.com1.z0.glb.clouddn.com/sample1.png)	
![](http://7xnby9.com1.z0.glb.clouddn.com/sample3.png)	
----------


若有问题，可及时联系：

QQ:452825089

mail:452825089@qq.com

blog:http://iceanson.github.io