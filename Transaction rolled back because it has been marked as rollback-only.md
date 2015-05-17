##Transaction rolled back because it has been marked as rollback-only
spring 具备多种事务传播机制，最常用的是REQUIRED，即如果不存在事务，则新建一个事务；如果存在事务，则加入现存的事务中。
示例代码如下：
```
public void A() {
    querySomething(...)；
    try {
      B()
    } catch () {
    }
    saveSomethinf()；
}

public void B() {
    throw Non-runtimeException()
}
```
此时B会和A存在一个事务中。如果B抛出异常没有捕获，即使在A中捕获并处理，仍会发生异常：**Transaction rolled back because it has been marked as rollback-only**
因为spring会在A捕获异常之前提前捕获到异常，并将当前事务设置为**rollback-only**，而A期望的行为是commit，当它发现状态为设置为rollback-only时，
则会抛出UnexpectedRollbackException
相关代码在AbstractPlatformTransactonManager.java中：
```
public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus);
			return;
		}
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus);
			// Throw UnexpectedRollbackException only at outermost transaction boundary
			// or if explicitly asked to.
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
			return;
		}

		processCommit(defStatus);
	}
```
**解决方法：**
在抛出异常的最原始地方处理异常，即在spring捕获到异常之前处理掉；如果需要抛出异常由上级方法处理，请捕获后抛出runtimeException
