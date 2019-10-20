---
layout: post
title: 使用springboot+mybatisplus开发评论功能
categories: 
 - java
 - springboot
permalink: /posts/4
nocomments: true  # Disable the comment box. This is EasyBook feature
---

使用java中Springboot+mybatisplus开发项目，是目前比较主流的项目开发手段，思路清晰明了。我在使用这项技术编写程序时，目前遇到逻辑比较复杂的项目需求莫过于获取文章、图片以及视频的评论。为了记录一下这个我当时对于这一需求的实现方法，我决定将它撰写成博客，既方便自己今后回顾，又可以与大家分享。

### 使用到的环境参数

1. 开发工具：IDEA
2. 基础工具：Maven+JDK8
3. 用到的技术：springboot+mybatisplus
4. 数据库：Mysql+Redis

### 准备工作

首先分析项目需求：

1. 我们需要将关于一篇文章的全部评论以分页格式的形式传送到前端
2. 前端可以通过调控参数决定按照时间顺序还是点赞数排行获取评论
3. 我们可以判断当前登陆的用户是否对某一条评论点过赞
4. 实现多级评论
5. 最终要的一点，这些需求需要在一个接口中实现

知道项目需求后，我们来看一下掘金网对于评论的编写模版：

```json
{
				"id": "5daace31f265da69f31e7f37",
				"content": "大胖哥😄😄😄",
				"userId": "5b857808e51d45387e51d2dd",
				"respUser": "5788b15cc4c971005ed26744",
				"respComment": "",
				"userInfo": {
					"objectId": "5b857808e51d45387e51d2dd",
					"username": "Z_CC",
					"avatarLarge": "https://user-gold-cdn.xitu.io/2019/3/9/1696265933aa6646?w=546&h=546&f=jpeg&s=342351",
					"selfDescription": "",
					"jobTitle": "前端工程师",
					"company": "",
					"viewedEntriesCount": 166,
					"collectedEntriesCount": 1,
					"level": 0,
					"isFollow": false
				},
				"respUserInfo": {
					"objectId": "5788b15cc4c971005ed26744",
					"username": "zz_jesse",
					"avatarLarge": "https://user-gold-cdn.xitu.io/2019/9/6/16d024e432ffb853?w=390&h=256&f=png&s=35937",
					"selfDescription": "公众号:前端张大胖,开源作品：SSR 开发骨架 Zz.js",
					"jobTitle": "fe http://zz.bigerfe.com",
					"company": "",
					"viewedEntriesCount": 2501,
					"collectedEntriesCount": 308,
					"level": 3,
					"isFollow": false
				},
				"likesCount": 0,
				"picList": [],
				"createdAt": "2019-10-19T08:49:53.885Z",
				"updatedAt": "2019-10-19T08:49:53.885Z",
				"subCount": 1,
				"replyCount": 1,
				"topComment": [{
					"id": "5dab175fe51d453536f15223",
					"content": "😄\u0000\u0000😃\u0000\u0000\u0000\u0000",
					"userId": "5788b15cc4c971005ed26744",
					"respUser": "5b857808e51d45387e51d2dd",
					"respComment": "5daace31f265da69f31e7f37",
					"userInfo": {
						"objectId": "5788b15cc4c971005ed26744",
						"username": "zz_jesse",
						"avatarLarge": "https://user-gold-cdn.xitu.io/2019/9/6/16d024e432ffb853?w=390&h=256&f=png&s=35937",
						"selfDescription": "公众号:前端张大胖,开源作品：SSR 开发骨架 Zz.js",
						"jobTitle": "fe http://zz.bigerfe.com",
						"company": "",
						"viewedEntriesCount": 2501,
						"collectedEntriesCount": 308,
						"level": 3,
						"isFollow": false
					},
					"respUserInfo": {
						"objectId": "5b857808e51d45387e51d2dd",
						"username": "Z_CC",
						"avatarLarge": "https://user-gold-cdn.xitu.io/2019/3/9/1696265933aa6646?w=546&h=546&f=jpeg&s=342351",
						"selfDescription": "",
						"jobTitle": "前端工程师",
						"company": "",
						"viewedEntriesCount": 166,
						"collectedEntriesCount": 1,
						"level": 0,
						"isFollow": false
					},
					"likesCount": 0,
					"picList": [],
					"createdAt": "2019-10-19T14:02:07.583Z",
					"updatedAt": "2019-10-19T14:02:07.583Z",
					"subCount": 0,
					"replyCount": 0,
					"topComment": null,
					"isLiked": false
				}],
				"isLiked": false
			}
```

参考一下这个例子，这是一个完整一个评论给模块，我们来分析一下：向前端传送的字段要包括，评论id、评论内容、评论人id、被评论人id、评论人详细信息（主要用来获取用户头像）、被评论人详细信息、该评论点赞数、当前用户是否点赞、评论时间以及更新时间。

分析完需要向前端传送什么，现在我们要分析前端需要给我们传哪些参数，我们才能把前端需要的评论给模版正确的传递给前端，具体参数如下：

```json
{
  "articleId": "String", 
  "pageNum": 0,
  "pageSize": 0,
  "type": "String",
  "userId": "String"
}
```

这些参数是必须由前端传送过来，我们才可以在数据库中找到相应数据，然后以正确的格式传送到前端。

### 具体执行过程

首先需要在ArticleCommentController中编写具体逻辑内容

```java
    /**
     * @param articleIdFormPage
     * @return
     */
    @ApiOperation("按时间，楼层，点赞数查询评论")
    @PostMapping("/list")
    public ServerResponse selectCommentByTimePage(@RequestBody ArticleIdFormPage articleIdFormPage){

        String userId = articleIdFormPage.getUserId();
        String articleId = articleIdFormPage.getArticleId();
        Integer pageNum = articleIdFormPage.getPageNum();
        Integer pageSize = articleIdFormPage.getPageSize();
        String type = articleIdFormPage.getType();

        if (articleId != null || pageNum != null || pageSize != null || type != null){
            if (type.equals("new")){
                #写selectAllPage接口
                IPage<ArticleComment> articleCommentIPageNew = articleCommentService.selectAllPage(articleId,pageNum,pageSize);
                PageVO<SimpleArticleCommentVO> simpleArticleCommentVOPageVO = selectComment(articleCommentIPageNew,userId);
                return ServerResponse.createBySuccess("按时间顺序分页查询全部评论成功！",simpleArticleCommentVOPageVO);
            } else if (type.equals("floor")){
                IPage<ArticleComment> articleCommentIPageFloor = articleCommentService.selectAllDescPage(articleId,pageNum,pageSize);
                PageVO<SimpleArticleCommentVO> articleCommentIPage = selectComment(articleCommentIPageFloor,userId);
                return ServerResponse.createBySuccess("按楼层数分页查询评论成功！",articleCommentIPage);
            } else if (type.equals("support")){
                IPage<ArticleComment> articleCommentIPageSupport = articleCommentService.selectAllBySupportPage(articleId,pageNum,pageSize);
                PageVO<SimpleArticleCommentVO> articleComments = selectComment(articleCommentIPageSupport,userId);
                return ServerResponse.createBySuccess("按点赞数分页查询评论成功！",articleComments);
            } else {
                return ServerResponse.createByErrorMessage("查询评论失败！");
            }

        } else {
            return ServerResponse.createByErrorMessage("分页查询评论失败！");
        }
```

然后，在ArticleCommentService中编写selectAllPage接口：

```java
    /**
     * @param articleId
     * @param pageNum
     * @param pageSize
     * @return
     */
    IPage<ArticleComment> selectAllPage(String articleId, Integer pageNum, Integer pageSize);
```

接着，在ArticleCommentServiceImpl中编写具体获取评论的方法：

```java
    /**
     * @param articleId
     * @param pageNum
     * @param pageSize
     * @return
     */
    @Override
    public IPage<ArticleComment> selectAllPage(String articleId, Integer pageNum, Integer pageSize) {
        try {
            Page<ArticleComment> page = new Page<>(pageNum,pageSize);
            #分页查询
            IPage<ArticleComment> articleComments = articleCommentMapper.selectPage(page,
                    new QueryWrapper<ArticleComment>()
                            .eq("status",1)
                            .eq("article_id",articleId)
                            .orderByDesc("create_time"));
            return articleComments;
        } catch (Exception e){
            e.printStackTrace();
            throw new RuntimeException(e.getMessage());
        }
    }
```

这时，正常情况来说，我们已经获取到了我们需要的评论内容，但是这个需求的难处，就是按照前端需要的格式返回给前端，这个格式是需要我们调整的，调整格式的代码我们也要在ArticleCommentController中完成，但是要搭配三个控制格式的实体类：

```java
    /**
     * 按一定格式返回前端评论信息
     * @author haoyingkai
     * @param articleCommentIPage
     * @return
     */
    public PageVO<SimpleArticleCommentVO> selectComment(IPage<ArticleComment> articleCommentIPage, String currentUserId){

        //创建PageVO对象
        PageVO<SimpleArticleCommentVO> simpleArticleCommentVOPageVO = new PageVO<>();
        simpleArticleCommentVOPageVO.setTotal(articleCommentIPage.getTotal());
        simpleArticleCommentVOPageVO.setSize(articleCommentIPage.getSize());
        simpleArticleCommentVOPageVO.setPageCounts(articleCommentIPage.getPages());
        simpleArticleCommentVOPageVO.setCurrentPage(articleCommentIPage.getCurrent());

        List<SimpleArticleCommentVO> simpleArticleCommentVOS = toVOList(articleCommentIPage.getRecords(), SimpleArticleCommentVO.class);

        List<SimpleArticleCommentVO> list = new ArrayList<>();
        for (SimpleArticleCommentVO simpleArticle:simpleArticleCommentVOS) {

            System.out.println(simpleArticle.toString());
            String userId = simpleArticle.getUserId();

            //创建SimpleArticleComment对象
            SimpleArticleCommentVO<SimpleUserVO, SimpleReplyVO> simpleUserVOSimpleArticleCommentVO = new SimpleArticleCommentVO<>();
            //在user表中查找用户信息
            List<User> userInfo = userService.getUserInfo(userId);
            Article article = articleService.getById(simpleArticle.getArticleId());

            if (currentUserId == null){
                simpleUserVOSimpleArticleCommentVO.setHasSupport(false);
            } else {
                Boolean hasSupport = commentSupportService.hasSupport(simpleArticle.getId(),currentUserId);
                if (hasSupport == true){
                    simpleUserVOSimpleArticleCommentVO.setHasSupport(true);
                } else {
                    simpleUserVOSimpleArticleCommentVO.setHasSupport(false);
                }
            }

            simpleUserVOSimpleArticleCommentVO.setCreateTime(simpleArticle.getCreateTime());
            simpleUserVOSimpleArticleCommentVO.setSupportCount(simpleArticle.getSupportCount());

            simpleUserVOSimpleArticleCommentVO.setReplyNum(simpleArticle.getReplyNum());
            simpleUserVOSimpleArticleCommentVO.setUserId(simpleArticle.getUserId());
            simpleUserVOSimpleArticleCommentVO.setArticleId(simpleArticle.getArticleId());
            simpleUserVOSimpleArticleCommentVO.setContent(simpleArticle.getContent());
            simpleUserVOSimpleArticleCommentVO.setId(simpleArticle.getId());
            simpleUserVOSimpleArticleCommentVO.setStatus(simpleArticle.getStatus());
            simpleUserVOSimpleArticleCommentVO.setFloor(simpleArticle.getFloor());
            //将userInfo转换为SimpleUserVO
            List<SimpleUserVO> simpleUserVOS = toVOList(userInfo, SimpleUserVO.class);
            simpleUserVOSimpleArticleCommentVO.setUserInfo(simpleUserVOS.get(0));

            String commentID = simpleArticle.getId();
            //在评论回复表中查询是否有评论回复
            List<ArticleCommentReply> articleCommentReplies = articleCommentReplyService.selectByCommentId(commentID);

            List<SimpleReplyVO> simpleReplyVOS = toVOList(articleCommentReplies, SimpleReplyVO.class);

            List<SimpleReplyVO> replyVOList = new ArrayList<>();
            for (SimpleReplyVO simpleReply:simpleReplyVOS) {

                String replyUserId = simpleReply.getUserId();
                List<User> replyUserInfo = userService.getUserInfo(replyUserId);
                List<User> respUserInfo = userService.getUserInfo(simpleReply.getRespUserId());

                SimpleReplyVO<SimpleUserVO,SimpleUserVO> simpleUserVOSimpleReplyVO = new SimpleReplyVO<>();
                simpleUserVOSimpleReplyVO.setCommentId(simpleReply.getCommentId());
                simpleUserVOSimpleReplyVO.setReplyCommentId(simpleReply.getReplyCommentId());
                simpleUserVOSimpleReplyVO.setContent(simpleReply.getContent());
                simpleUserVOSimpleReplyVO.setId(simpleReply.getId());
                simpleUserVOSimpleReplyVO.setStatus(simpleReply.getStatus());
                simpleUserVOSimpleReplyVO.setRespUserId(simpleReply.getRespUserId());
                simpleUserVOSimpleReplyVO.setUserId(simpleReply.getUserId());
                simpleUserVOSimpleReplyVO.setCreateTime(simpleReply.getCreateTime());

                List<SimpleUserVO> simpleUserVOS1 = toVOList(replyUserInfo, SimpleUserVO.class);
                List<SimpleUserVO> simpleUserVOS2 = toVOList(respUserInfo, SimpleUserVO.class);
                System.out.println(simpleUserVOS1.toString());
                simpleUserVOSimpleReplyVO.setUserInfo(simpleUserVOS1.get(0));
                simpleUserVOSimpleReplyVO.setRespUserInfo(simpleUserVOS2.get(0));

                replyVOList.add(simpleUserVOSimpleReplyVO);
           }
            simpleUserVOSimpleArticleCommentVO.setTopCommentList(replyVOList);

            //将simpleUserVOSimpleArticleCommentVO转换为列表
            list.add(simpleUserVOSimpleArticleCommentVO);
            System.out.println(simpleUserVOSimpleArticleCommentVO);
            System.out.println(list.toString());
        }


        simpleArticleCommentVOPageVO.setCommentList(list);
        return simpleArticleCommentVOPageVO;
    }
}
```

我们利用到三个类来调整格式：

实体类一PageVO：

```java
import lombok.Data;

import java.util.List;

@Data
public class PageVO<T> {

    private Long total;

    private Long currentPage;

    private Long size;

    private Long pageCounts;

    private List<T> commentList;

}
```

这个类里面包含了关于分页相关的信息，我们要在commentList中放置评论信息。

实体类二SimpleArticleCommentVO：

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.Data;

import java.util.Date;
import java.util.List;

@Data
public class SimpleArticleCommentVO<S> {

    private String id;

    private String articleId;

    private String UserId;

    private String content;

    /**
     * 回复个数
     */
    private Integer replyNum;

    /**
     * 赞成数
     */
    private Integer supportCount;

    private Integer status;

    private Integer floor;

    private boolean hasSupport;

    #如果不对时间进行格式调整，输出的时间永远比实际差8小时
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm", timezone = "GMT+8")
    private Date createTime;

    private SimpleUserVO userInfo;

    private List<S> topCommentList;

}
```

这个类里包含了所有评论的信息，我们需要把评论回复放到topCommentList中。

实体类三SimpleReplyVO：

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import lombok.Data;

import java.time.LocalDateTime;

@Data
public class SimpleReplyVO {

    private String id;

    private String commentId;

    private String replyCommentId;

    /**
     * 回复对象id
     */
    private String respUserId;

    /**
     * 本人id
     */
    private String userId;

    /**
     * 评论回复内容
     */
    private String content;

    private Integer status;

    @JsonDeserialize(using = LocalDateTimeDeserializer.class)
    @JsonSerialize(using = LocalDateTimeSerializer.class)
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm", timezone = "GMT+8")
    private LocalDateTime createTime;

    private SimpleUserVO userInfo;

    private SimpleUserVO respUserInfo;

}

```

这里面需要放置的是回复评论的内容。

### 结果

编写到这里，我们就算基本结束了，我们测试一下代码。

输入参数如下：

```json
{
  "articleId": "7863650274253668352",
  "pageNum": 0,
  "pageSize": 2,
  "type": "new",
  "userId": "574531004786540544"
}
```

输出评论字段如下：

```
 {"status": 200,
  "msg": "按时间顺序分页查询全部评论成功！",
  "data": {
    "total": 2,
    "currentPage": 1,
    "size": 2,
    "pageCounts": 1,
    "commentList": [
      {
        "id": "82b93948b137f0021ad6ffb99f0649ea",
        "articleId": "7863650274253668352",
        "content": "特别的积极",
        "replyNum": 0,
        "supportCount": -2,
        "status": 1,
        "floor": 2,
        "hasSupport": false,
        "createTime": "2019-10-20 09:59",
        "userInfo": {
          "id": "560924008665579520",
          "phone": "15333617373",
          "email": null,
          "name": "郝英凯",
          "signature": "这个人很低调,什么都没说!",
          "image": "https://***.oss-cn-beijing.aliyuncs.com/yunding/20190906/1c8e4b7a75e04774a5afde56e6f5207a-image",
          "sex": "男",
          "job": null,
          "address": null
        },
        "topCommentList": [],
        "userId": "560924008665579520"
      },
      {
        "id": "f68fb1670e6aeea2818b7a6c64c88227",
        "articleId": "7863650274253668352",
        "content": "很不错",
        "replyNum": 1,
        "supportCount": 0,
        "status": 1,
        "floor": 1,
        "hasSupport": false,
        "createTime": "2019-10-20 09:58",
        "userInfo": {
          "id": "560924008665579520",
          "phone": "15333617373",
          "email": null,
          "name": "郝英凯",
          "signature": "这个人很低调,什么都没说!",
          "image": "https://***.oss-cn-beijing.aliyuncs.com/yunding/20190906/1c8e4b7a75e04774a5afde56e6f5207a-image",
          "sex": "男",
          "job": null,
          "address": null
        },
        "topCommentList": [
          {
            "id": "edbc518883f952357ab4773f4a6a4405",
            "commentId": "f68fb1670e6aeea2818b7a6c64c88227",
            "replyCommentId": "f68fb1670e6aeea2818b7a6c64c88227",
            "respUserId": "560924008665579520",
            "userId": "560924008665579520",
            "content": "非常好",
            "status": 1,
            "createTime": "2019-10-20 09:58",
            "userInfo": {
              "id": "560924008665579520",
              "phone": "15333617373",
              "email": null,
              "name": "郝英凯",
              "signature": "这个人很低调,什么都没说!",
              "image": "https://***.oss-cn-beijing.aliyuncs.com/yunding/20190906/1c8e4b7a75e04774a5afde56e6f5207a-image",
              "sex": "男",
              "job": null,
              "address": null
            },
            "respUserInfo": {
              "id": "560924008665579520",
              "phone": "15333617373",
              "email": null,
              "name": "郝英凯",
              "signature": "这个人很低调,什么都没说!",
              "image": "https://***.oss-cn-beijing.aliyuncs.com/yunding/20190906/1c8e4b7a75e04774a5afde56e6f5207a-image",
              "sex": "男",
              "job": null,
              "address": null
            }
          }
        ],
        "userId": "560924008665579520"
      }
    ]
  }
}
```

回执中给到了两条评论，一条是一级评论，没有评论回复，还有一条是多级评论，topCommentList中有二级评论的内容。至此，我们基本上可以还原最开始找到的评论的示例，评论功能算是写完了。

虽然写博客很快，但是但是敲代码的时候费了好大劲，希望这段关于评论的代码可以帮助大家对于编写获取评论有一条清晰的方向，代码仅供大家参考。