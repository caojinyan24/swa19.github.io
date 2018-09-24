---
layout: post
title:  "Java开发之事务管理"
date:   2018-01-16 20:47:52 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

前几天线上出现了一个问题.在做一个操作的时候,发现一条数据的状态值不对.

~~~
	@Override
	@Transactional(noRollbackFor = CollectionException.class)
	public int change(String[] businessCenters, String staffNo) {
		List<String> lst_caseIds = caseAssignMapper.getCaseIdsByBusinessCenters(businessCenters);
		if(!CollectionUtils.isEmpty(lst_caseIds)) {
			Object savePoint = null;
			boolean lockOverdueCaseSuccess = false;
			String[] arr_caseIds = lst_caseIds.toArray(new String[lst_caseIds.size()]);
			int effectCount = lst_caseIds.size();
			try{
				savePoint = TransactionAspectSupport.currentTransactionStatus().createSavepoint();
				lockOverdueCaseSuccess = overdueCaseAssignLockService.lockOverdueCaseByIds(arr_caseIds);
				if(!lockOverdueCaseSuccess) {
					throw new CollectionException("案件状态已变更,请重试!");
				}
				effectCount = caseAssignMapper.changeForecast2CallList(lst_caseIds);				
			} catch (Exception e) {
				TransactionAspectSupport.currentTransactionStatus().rollbackToSavepoint(savePoint);
				throw new CollectionException(e);
			}finally{
				if(lockOverdueCaseSuccess) {
					overdueCaseAssignLockService.unLockOverdueCaseByIds(arr_caseIds);
				}
			}

			return effectCount;
		}
		return 0 ;
	}
~~~

~~~
	//更新状态lock值为1
     @Override
	@Transactional(propagation=Propagation.REQUIRES_NEW)
	public boolean lockOverdueCaseByIds(String[] arr_caseIds) {
		return caseAssignMapper.lockCasesBeforeAssign(arr_caseIds,lockableCdns);		
	}
	//更新状态值lock为0
	@Override
	@Transactional
	public boolean unLockOverdueCaseByIds(String[] arr_caseIds) {		
		 return likaAssignMapper.unLockCasesBeforeAssignByLK(tempCaseIdList);			
	}
~~~

在操作的时候,发现某条数据的lock的值为1,且这个状态值只有在上边的两个方法中得到更新,而检查这两个方法的日志后发现,每次更新成1的数据量和更新成0的数据量一致,且不存在异常日志.
仔细检查数据,发现发现这条异常的数据在做这个操作的同时做了另一个操作:

~~~
    @Override
    @Transactional
    public void doCallSummary(InboxCallSummaryDto record) {
        OverDueCaseData overDueCaseData = this.queryOverdueCaseByIds(Lists.<Long>newArrayList(record.getCaseId()));
        InboxCallSummary callSummary = new InboxCallSummary();
        BeanUtils.copyProperties(record, callSummary);
        overduecase.setLastDealTime(new Date());
        overduecase.setFirstDealTime(new Date());
        overduecase.setLastCallNo(record.getCallNo());
        this.updateOverDueCase(overduecase);

        }
      
    }
~~~

理一下时间点:
11:02:19.422   change.lockOverdueCaseByIds
11:02:21.470   doCallSummary.queryOverdueCaseByIds
11:02:40.0     change.unLockOverdueCaseByIds
11:02:41.0	doCallSummary.updateOverDueCaseByDataSource

所以据此可以定位问题原因为:更新前读到的不是最新数据,导致最新数据被覆盖.

那么怎么解决这个问题呢?首先考虑的是要加锁,保证同时只有一个线程对数据做更新操作;另一个是更新之前再次读取数据.
所以做出的更改为,将`queryOverdueCaseByIds`的查询sql改为`selectforupadte`.
当然,以上的代码做了很多简化,实际需要解决的问题不单单是对select操作加锁的问题,以下总结下这次排查过程中重新梳理的知识点:

1. spring类内的方法调用导致事务不生效
selectforupdate会对选定的数据加锁,且直到事务提交或回滚的时候才会释放锁.但是测试过程中发现添加之后这个锁并不会释放,导致测试的另一个线程一直被阻塞.检查代码后发现存在类内的方法互相调用的情况.
也就是A方法和B方法同属于一个类中,A调用B,而B方法上添加了@Transactional注解.这个时候这个事务是不生效的.

2. mysql默认的事务隔离级别是readRepeatable
可以避免脏读和不可重复读,但可能会引起幻读.

找资料的时候看到的一个介绍很详细的关于spring事务管理的介绍:`http://blog.csdn.net/trigl/article/details/50968079`.所以就不当搬运工了

这个问题就告一段落,需要说明的是,数据库的设计很重要,尤其是表设计中的create_time和update_time最好不要有业务含义,而且update_time最好设计成数据库自动更新的,对查找问题很有帮助.