# Concept
在新的Jetpack Compose Navigation下，其实已经没有了Fragment的概念了。或者说Jetpack Compose其实已经不需要Fragment的概念了，如果极端一些整个应用可以只需要一个Activity。  
所以Jetpack Compose Navigation主要就是从一个Composable方法导航到另一个Composable方法
# Base
基于路由（Route）和返回栈（BackStack）原理，通过 NavController 管理页面状态，NavHost 将路由映射到具体的 Composable 函数。  
当路由发生变化时，Compose 通过重组（Recomposition）更新 UI。
## NavHost
声明式导航，是一个容器，通过路由标识符（Route）将路径与 Composable 界面映射起来。用户导航时，NavHost 根据当前目的地（Destination）查找并显示相应的 Composable 函数。
## NavController
状态管理与回退栈，负责记录和管理页面跳转历史。当执行导航动作时，新的 Destination 被压入栈顶（Push），点击返回时将其弹出（Pop），实现了单例模式的栈管理。
## Route
路由机制，每个页面是一个唯一定义的 URL 风格的字符串（例如 "user/123"），NavController 基于此字符串来查找和替换当前显示的屏幕。  
## NavGraphBuilder
使用 composable("route") { ... } 语法定义每个路由对应的界面
## 重组替代页面切换
与传统使用 Fragment 替换不同，Compose Navigation 在状态（即当前路由）改变时，触发 Composable 函数的重组，以显示新界面，这种方式内存效率更高。
## 数据共享 (ViewModel)
配合导航，可以使用单例 ViewModel 或结合路由传递参数（arguments），在不同 Composable 之间共享数据，确保返回时状态不丢失。 
