---
layout: post
title: flyBird - 使用 Scala 和 JavaFx 动画实现
categories:
  - Java
  - Scala
  - JavaFx
  - FlappyBird
---

> 本文介绍了使用 JavaFx 和 Scala 实现的 FlappyBird 游戏。核心内容包括重力下落、按键飞行、自动前进、碰撞检测。主要使用的技术是 —— JavaFx 的动画，没错，就是动画实现的游戏引擎，以及组件和嵌套组件的属性和值绑定：比如飞行和下落事件由动画执行，绑定值，再绑定不同颜色。

当然，动画不是拿来做这个的，JavaFx 2D 也不是拿来做游戏的 —— 这个 DEMO 主要讲解了动画做游戏的限制，比如所有状态统一管理，碰撞检测必须让被检测碰撞的组件放在一个组内 —— 因为所有的坐标都是相对于父元素/自身偏移计算的。但是，总而言之，100 行代码实现的 FlyBird，能够看出 JavaFx Builder 模式以及其属性和值绑定的简洁优雅。

<img src='http://static2.mazhangjing.com/20190320/91c7c42_2019-03-2017.26.21.gif' style='margin-left:0px'>

```scala
class FlyBird extends Application {

  //组件和嵌套组件
  val bird = new Rectangle(100, 200, 20,20)
  val walls: List[Node] = List[Node]()
  val root: Parent = {
    val group = new Group()
    0.to(10).foreach(i => {
      val s = i * 100 * 6
      val topWall = new Rectangle(300 + s, 0, 20, 100)
      val bottomWall = new Rectangle(400 + s,300,20,100)
      val topWall2 = new Rectangle(500 + s,0,20,150)
      val bottomWall2 = new Rectangle(600 + s,250,20,150)
      val topWall3 = new Rectangle(700 + s,0,20,150)
      val bottomWall3 = new Rectangle(800 + s,250,20,150)
      walls ++= Array(topWall, bottomWall, topWall2, bottomWall2, topWall3, bottomWall3)
    })
    group.getChildren.add(bird)
    group.getChildren.addAll(walls:_*); group
  }
  val scene = new Scene(root, 700, 400)

  //属性和状态
  private val movePixel = new SimpleDoubleProperty(0.0)
  private val flying = new SimpleBooleanProperty(false)

  private val animate: Timeline = {
    val t = new Timeline(
      new KeyFrame(Duration.millis(50), _ => {
        if (doDetectedAction) { doEndGameAction() }
        walls.foreach(node => node.setTranslateX(node.getTranslateX - 5))
        if (!flying.get()) movePixel.set(movePixel.get() + 3)
        else movePixel.set(movePixel.get() - 10)
      }))
    t.setAutoReverse(false)
    t.setCycleCount(-1); t
  }

  private def initBindAction(): Unit = {
    //为按键绑定移动量，bird 和移动量绑定
    scene.setOnKeyPressed(e => e.getCode match {
      case KeyCode.SPACE => flying.set(true)
      case KeyCode.S => doStartGameAction()
      case _ =>
    })
    scene.setOnKeyReleased(e => e.getCode match {
      case KeyCode.SPACE => flying.set(false)
      case _ =>
    })
    bird.translateYProperty().bind(movePixel)
    import com.mazhangjing.sfx.Utils._
    //为 flying 绑定 bird 的颜色变化
    flying.addListener {
      if (flying.get()) bird.setFill(Color.BLUE)
      else bird.setFill(Color.RED)
    }
  }

  private def doDetectedAction: Boolean = {
    walls.exists(node => 
      bird.getBoundsInParent.intersects(node.getBoundsInParent))
  }

  def doStartGameAction(): Unit = {
    animate.playFromStart()
  }

  def doEndGameAction(): Unit = {
    animate.stop()
    movePixel.set(0)
    walls.foreach(node => node.setTranslateX(0))
  }

  override def start(stage: Stage): Unit = {
    stage.setTitle("Fly Bird")
    stage.setScene(scene)
    initBindAction()
    stage.show()
  }
}
```

注意，这里的 Utils 类隐式转换的内容如下 —— 这里提供了一种更为自然的语法糖，当然，使用原始的 `(e,o,n) => op` 和 `o => op` 来表示值变化和值更新监听器也可以。

```scala
implicit class SuperProperty[T](property: Property[T]) {
  def addListener(op: => Unit): Unit = {
    val changedListener: ChangeListener[T] = (e, o, n) => op;
    property.addListener(changedListener)
  }
}
```

-----

2019-03-20 撰写本文

2019-03-23 更新了一个在 foreach 中强行 return 的风格问题，现在使用 exists 方法。更新了一个措辞上的问题。
