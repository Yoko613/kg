# kg
知识图谱项目代码

```

/*eslint-disable */
/**
 * @file 题库试题解析
 * @author liuchang33@baidu.com
 */

function OnlineError(args) {
    // 题目原始数据
    this.data = args || {};

    if (typeof this.data.bdjson == 'undefined' || !this.data.bdjson) {
        return;
    }
    this.bdjson = JSON.parse(this.data.bdjson);
    this.que_info = this.bdjson.que_info;
    this.queStem = this.bdjson.que_stem;
    if (typeof this.queStem == 'undefined') {
        this.queStem = [{c:'题干为空哦，请点击查看试题详情',t: 'p'}];
    }
    // 模板
    this.tpl = doT.template(__inline('dbjson.tpl'));
    // 判断试题解析字段
    this.analysis = this.analysis();
    // 是否含有图片
    this.hasImg = this.hasImg();
    // 是否全是图片【判断多余】
    // this.hasAllImg = this.hasAllImg();
    if (this.hasImg) {
        this.resultData = this.calHeight();
    }
    else{
        // 没有图片的题目只取第一段
        this.resultData = this.queStem[0];
    }
    this.init();
}

OnlineError.prototype = {
    // 初始化
    init: function () {
        var parsedData = new parseQuestion();
        var title = parsedData.parseBdjson(this.resultData);

        var list = {
            uri: this.data.uri,
            ask: this.analysis,
            subject: this.data.subject,
            year: this.data.year,
            analysis: this.data.analysis,
            title: title,
            num: this.data.num
        };
        // 渲染模板
        this.html = $(this.tpl(list));
        $('.txt').append(this.html);
    },
    // 试题解析字段
    analysis: function () {
        var oThis = this;
        var analysis;
        if (oThis.que_info.type == 'multi'
              && oThis.que_info.sub_type == 'cloze'){// 英语完型填空题
            analysis = '填空题';
        }
        else if (oThis.que_info.type == 'multi'
              && oThis.que_info.sub_type == 'mbk'){// 语法填空题
            analysis = '填空题';
        }
        else if (oThis.que_info.type == 'multi'
              && oThis.que_info.sub_type == 'cbk'){// 补全信息题
            analysis = '信息题';
        }
        else if (oThis.que_info.type == 'multi'){// 综合题（一大题多小题）
            analysis = '综合题';
        }
        else if (oThis.que_info.type == 'sc') {// 单选
            analysis = '单选';
        }
        else if (oThis.que_info.type == 'mc') {// 多选
            analysis = '多选';
        }
        else if(oThis.que_info.type == 'qa' && oThis.que_info.sub_type == 'composition'){// 作文
            analysis = '作文';
        }
        else if (oThis.que_info.type == 'qa') {// 问答题
            analysis = '问答题';
        }
        else if (oThis.que_info.type == 'bk') {// 填空题
            analysis = '填空题';
        }
        return analysis;
    },

    // 题目标题中是否含有图片
    hasImg: function () {
        var oThis = this;
        var hasImg = false;
        $.each(oThis.queStem, function (index, item) {
            if (hasImg) {
                return false;
            }
            if (item.t === 'img') {
               hasImg = true;
               return false;
            }
            else if(item.c && item.c.length > 0 && $.isArray(item.c)) {
                $.each(item.c, function (index, item2) {
                    if (item2.t === 'img') {
                       hasImg = true;
                       return false;
                    }
                })
            }
            else if (item.c && item.c.length > 0 && !$.isArray(item.c)) {
                 if (item.t === 'img') {
                       hasImg = true;
                       return false;
                    }
            }
        });
        return hasImg;
    },
    // 题目标题中是否全是图片
    hasAllImg: function () {
        var oThis = this;
        var hasImg = true;
        $.each(oThis.queStem, function (index, item) {
            if (!hasImg) {
                return false;
            }
            if (item.t === 'img') {
               hasImg = true;
               return false;
            }

            if(item.c && item.c.length > 0 && $.isArray(item.c)) {
                $.each(item.c, function (index, item2) {
                    if (item2.t === 'img') {
                       hasImg = true;
                    }
                    else {
                        hasImg = false;
                        return false;
                    }
                })
            }
            else if(item.c && item.c.length > 0 && !$.isArray(item.c)) {
                if (item.t === 'img') {
                   hasImg = true;
                   return false;
                }
            }
        });
        return hasImg;
    },
    // 计算高度(无图片时不超过三行)
    calHeight: function () {
        var oThis = this;
        var ellipsisContainer = $('.layout');
        // // 清空上次的内容
        ellipsisContainer.html('');
        window.lmaxHeight = 80;
        // 存放最终没超3行高度的结果
        window.resultData = [];
        window.hasOverHeight = false;
        $.each(oThis.queStem, function (index, item) {
            window.index = index;
            var obj = cloneObject(item);
            var queParse = new parseQuestion();
            // 如果没有子只有一层
            if ((item.c && $.isArray(item.c) && item.c.length == 1)
                || (item.c && !$.isArray(item.c))) {

                var str = queParse.parseBdjson(item);
                ellipsisContainer.append($(str));
                var curHeight = ellipsisContainer.height();
                if (curHeight < window.lmaxHeight) {
                    obj = item;
                }
                else {
                    // 全是文字的话都放进去
                    if (!$.isArray(item.c) && item.t != "img" && window.index < 1) {
                        window.resultData.push(item);
                    }
                    if ($.isArray(item.c) && item.c.length == 1 && item.t != "img" && window.index < 1) {
                        window.resultData.push(item);
                    }
                    oThis.hasOverHeight = true;
                    return false;
                }

            }
            else if (item.c && $.isArray(item.c) && item.c.length > 0) {
                obj.c = [];
                if (item.t === 'p') {
                   ellipsisContainer.append($('<p></p>'));
                }
                $.each(item.c, function (index2, item2) {
                    var str = queParse.parseBdjson(item2);
                    ellipsisContainer.find('p').append($(str));
                    var curHeight = ellipsisContainer.height();
                    if (curHeight < window.lmaxHeight) {
                        obj.c.push(item2);
                    }
                    else {
                        // 全是文字的话都放进去
                        if (!$.isArray(item2.c) && item2.t != "img") {
                            if (window.index < 1) {
                                obj.c.push(item2);
                            }
                            else {
                                obj.t = "span";
                            }
                        }
                        else {
                            if (window.index < 1) {
                                obj.c.push(item2);
                            }
                        }
                        oThis.hasOverHeight = true;
                        return false;
                    }

                });
            }
            window.resultData.push(obj);
        });
        return window.resultData;
    }
};
```
