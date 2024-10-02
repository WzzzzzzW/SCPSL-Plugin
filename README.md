# <div align="center">Wzzzzzz-Plugin</div>
<div align="center">一个基于Exiled编写的SCPSL的游戏插件</div>

# 项目介绍
在写这个插件的同时，也是我第一次接触到C#的项目编写，与以往接触相比最大的不同是，这个项目是从头开始写的，不像曾经那样只是在别人的源代码上修修改改吧
# 项目编写环境
* 环境:Visual Studio 2022
* 编程语言:C#
# 项目使用方法
* <b>安装 Visual Studio Code</b>
  * 访问官网下载 (<a href="https://visualstudio.microsoft.com/zh-hans/vs/">点此下载</a>)
  * 打开Visual Studio installer
    
    ![image](https://github.com/user-attachments/assets/a6847ab3-637b-4f5a-9601-0a37b24bf088)
    
  * 安装对应版本(Community版本)
    
    ![image](https://github.com/user-attachments/assets/ae394759-35c9-44b8-a607-f62a3a4bdc37)

  * 选择工作复核(必要的)

    ![image](https://github.com/user-attachments/assets/71e5b00d-23cc-4c9c-a421-3d10d60ecb16)

  * 右下角点击开始安装

* <b>打开项目</b>
  * 解压好项目,找到.sln后缀的文件打开
    
   ![image](https://github.com/user-attachments/assets/b461cca3-c526-4c21-a97b-8eb0539906be)


# 注意事项
* <b>检查文件依赖</b>
  * 点击引用查看
    
  ![image](https://github.com/user-attachments/assets/1c26dc64-9978-4ccc-b39e-feff4cd2124d)

  * 如果遇到依赖缺失，请到下列目录自行添加

  ![image](https://github.com/user-attachments/assets/f9bab3b4-56d3-4f4e-ad86-02505d428ed1)

# 使用小手册（对部分代码解释）

* <h2>索引</h2>
  
| 类名  | 描述 |
| ------------- | ------------- |
| Config.cs  | 集中管理插件的地方  |
| ITEM.cs  | \  |
| 零碎的功能.cs  | \  |
| 人物比例.cs  | 基于对RemoteAdminCommandHandler(RA控制面板)的添加  |
|生成自义定物品.cs| 基于对RemoteAdminCommandHandler(RA控制面板)的添加| 


* <h2>Config.cs</h2>

  * 暂无文档

* <h2>ITEM.cs</h2>

  * <b>物体生成方法</b>
  
  ```c#
  ItemPickup itemPickup = ItemPickup.Create({物品ID},{Vector3坐标}, Quaternion.identity);///定义物品
  itemPickup.Transform.localPosition = Vector3 Vector3;///设置刷新位置
  itemPickup.GameObject.transform.localScale = Vector3.one * 12f;///设置物体沿xyz轴扩大12倍（放大12倍）
  itemPickup.Weight = 10f;///设置物体质量
  NetworkServer.Spawn(itemPickup.GameObject);///生成物体
  ```
* <h2>零碎的功能.cs</h2>
  
  * <b>捕获函数调用</b>
  
    * 利用此原理，可以在游戏执行某个函数时，插入自己想要执行的东西
    * 以下示例里，捕获的是`Respawning.RespawnEffectsController`类名下`ServerExecuteEffects`函数
    * 当捕获到的时候,判断刷新的阵营是否是混沌分裂者，如果利用'foreach'函数遍历所有玩家，给每位玩家发送6秒的广播信息，同时游戏内部大标题提示
  
  ```c#
  public sealed class Patch
  {
      [HarmonyPatch(typeof(Respawning.RespawnEffectsController), nameof(Respawning.RespawnEffectsController.ServerExecuteEffects))]
      public static void Postfix(Respawning.RespawnEffectsController __instance, ref SpawnableTeamType team)
      {
          Config config = new Config();
          if (team == SpawnableTeamType.ChaosInsurgency)
          {
              foreach(Exiled.API.Features.Player player in Exiled.API.Features.Player.List)
              {
                  player.Broadcast(6,config.CI);
              }
              Exiled.API.Features.Cassie.MessageTranslated("检测到不明组织进入设施", config.CI);
          }
      }
  }
  ```
  * <b>读取游戏内置预制体(Perfabs)</b>
  
    * 利用此原理，可以在游戏的任意一个地方添加武器柜，门，电板箱等等物品
    * 下列示例中，利用了`foreach`循环将所有预制体进行读取
    * 这些预制体利用字典类型存储，所以在创建的时候直接使用`NetworkClient.prefabs[i.Key]`
    * 另外，在生成的过程中，利用了延迟函数，以防创建过快导致游戏卡死
    * 具体内容可以自行去研究
  ```c#
   public IEnumerator<float> test()
   {
       
       Quaternion rot = Quaternion.Euler(0, 180, 0);
       int a = 0;
       foreach (var i in NetworkClient.prefabs)
       {
           Vector3 pos = new Vector3(-15 + 2 * a - 80f, 992, -42);
           Exiled.API.Features.Log.Info(i);
           if (a >= 45 && a<=75)
           {
               //GameObject benchPrefab = GameObject.CreatePrimitive(PrimitiveType.Plane);
               NetworkServer.Spawn(Object.Instantiate(NetworkClient.prefabs[i.Key], pos, rot));
               Exiled.API.Features.Log.Info(pos);
               yield return Timing.WaitForSeconds(0.7f);
           }
           a += 1;
       }
   }
  ```

* <h2>人物比例</h2>

  * 核心内容主要是通过内置的`Scale`方法进行修改人物比例
  * 该功能涉及到对RA控制面板添加指令的内容
  * 下列示例中`ICommand.Execute`函数的参数`sender`可以得到执行这段指令的管理员,必要时可以进行一些操作

```c#
 [CommandHandler(typeof(RemoteAdminCommandHandler))]
  public  class 人物比例 : ICommand, IUsageProvider
 {
     string ICommand.Command => "size"; ///指令的名字
 
     string[] ICommand.Aliases => new string[] { "size" };
 
     string ICommand.Description => "自义定人物比例大小";///指令的描述
 
     string[] IUsageProvider.Usage => new string[]
     {
         "玩家id，all为全体,不填则为自己(记得留空格)","x","y","z"///指令用法提示
     };
     bool ICommand.Execute(ArraySegment<string> arguments, ICommandSender sender, out string response)
     {
 
         float x = float.Parse(arguments.At(1));
         float y = float.Parse(arguments.At(2));
         float z = float.Parse(arguments.At(3));
         bool flag = arguments.At(0).ToString() ==null;
         if (flag)
         {
             Player player = sender as Player;
             ObjectSpawner.SpawnSchematic("", player.Position);
             response = "YES";
             return true;
         }
         if(arguments.At(0).ToString() == "all")
         {
             foreach(Player player1 in Player.List)
             {
                 player1.Scale = new Vector3(x, y, z);
                 
             }
             response = "Done！";
             return true;
         }
         else
         {
             Player player = Player.Get(int.Parse(arguments.At(0)));
             player.Scale = new Vector3(x, y, z);
             response = "Done！";
             return true;
         } 
     }
 }
```

  









