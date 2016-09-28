
不用锁的例子

## **比锁更好的用例**

银行转账, 使用锁的解决方案, 容易造成死锁

```scala
class BankAccount{
    private var balance = 0
    def deposit(amount: Int): Unit = this.synchronized {
        if (amount > 0) balance = balance + amount
    }
    def withdraw(amount: Int): Int = this.synchronized {
        if (0 < amount && amount <= balance){
            balance = balance - amount
            balance
        } else throw new Error("insufficient funds")
   }
}

def transfer(from: BankAccount, to: BankAccount, amount: Int): Unit = {
    from.synchronized {
        to.synchronized {
            from.withdraw(amount)
            to.deposit(amount)
        }
    }
}
```

问题在于 from, to 同时调用时会死锁, 即便使用 tryLock 这种做保护。即便是解决了死锁的问题, 锁本身也是比较耗资源的

```scala
object BankAccount {
  case class Deposit(amount: BigInt) {
    require(amount > 0)
  }
  case class Withdraw(amount: BigInt) {
    require(amount > 0)
  }
  case object Done
  case object Failed
}

class BankAccount extends Actor {

  import BankAccount._

  var balance = BigInt(0)

  def receive = LoggingReceive {
    //Deposit messages add amount to balance state
    case Deposit(amount) =>
      balance += amount
      sender ! Done

    //Withdraw messages subtract amount from balance state
    case Withdraw(amount) if amount <= balance =>
      balance -= amount
      sender ! Done

    //Any other message would return a failure to the sender
    case _ => sender ! Failed
  }
}

per request actor
object WireTransfer {
  case class Transfer(from: ActorRef, to: ActorRef, amount: BigInt)
  case object Done
  case object Failed

  //actor implementing the actions of a wire transfer between two bank account actors
  class WireTransfer extends Actor {
    import WireTransfer._
    def receive = LoggingReceive {
      //If Transfer message is received, send withdraw message to 'from' and wait for reply
      case Transfer(from, to, amount) =>
        from ! BankAccount.Withdraw(amount)
        context.become(awaitFrom(to, amount, sender))
    }
  
    //If Withdraw was successful, send deposit to other bank account actor, or else give them a failure message
    def awaitFrom(to: ActorRef, amount: BigInt, customer: ActorRef): Receive = LoggingReceive {
      case BankAccount.Done =>
        to ! BankAccount.Deposit(amount)
        context.become(awaitTo(customer))
      case BankAccount.Failed =>
        customer ! Failed
        context.stop(self)
    }
  
    //If deposit was successful, send 'Done' to original actor that sent Transfer message
    def awaitTo(customer: ActorRef): Receive = LoggingReceive {
      case BankAccount.Done =>
        customer ! Done
        context.stop(self)
    }
  }

class TransferMain extends Actor {

  //First create two BankAccount actors
  val accountA = context.actorOf(Props[BankAccount], "accountA")
  val accountB = context.actorOf(Props[BankAccount], "accountB")

  //send a deposit message to accountA
  accountA ! BankAccount.Deposit(100)

  //If a 'Done' message is received back, call a transfer function
  def receive = LoggingReceive {
    case BankAccount.Done => transfer(70)
  }

  //transfer function creates a transacton actor and sends a 'Transfer' message to it between
  //accountA and accountB for the specified amount.
  def transfer(amount: BigInt): Unit = {

    val transaction = context.actorOf(Props[WireTransfer], "transfer")

    transaction ! WireTransfer.Transfer(accountA, accountB, amount)

    context.become(LoggingReceive {
      case WireTransfer.Done =>
        println("successs")
        context.stop(self)
      case WireTransfer.Failed =>
        println("failed")
        context.stop(self)
    })
  }}}
```