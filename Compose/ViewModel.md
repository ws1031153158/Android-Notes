# 数据共享
不同方式生命周期不同，若不同地方各自创建则为多个实例。
## 手动传递
父子关系明确，层级不深。  
例如：  
val vm: XXViewModel = viewModel()（viewmodel存储在viewModelStore中，同一个 ViewModelStore + 同一个 key 拿到的是同一个实例）往下传递。
## activityViewModels
Activity级别共享，整个App都需要共享的状态，如用户登录状态、主题设置，在任意 Fragment/Composable 里，都能拿到同一个 Activity 级别的 ViewModel。  
例如：  
//Activity中  
val vm: XXViewModel by viewModels()  
//Composable 中  
@Composable  
fun AnyScreen() {  
    val activity = LocalContext.current as ComponentActivity  
    val vm: XXViewModel = viewModel(activity)  
}
## Navigation 共享
同一个 NavGraph 内的多个页面共享，在 NavGraph 内任意页面获取，如多步骤表单共享数据、主从页面等。  
例如：  
@Composable  
fun SomeScreen(  
    navController: NavController  
) {  
    // 获取 NavGraph 级别的 ViewModel  
    val backStackEntry = remember(navController) {  
        navController.getBackStackEntry(Screen.XX.route)  
    }  
    val vm: XXViewModel = viewModel(backStackEntry)  
}
