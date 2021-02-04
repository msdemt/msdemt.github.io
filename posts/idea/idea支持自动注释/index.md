# Idea支持自动注释


1. idea支持新建类自动添加注释  
    File-Settings-Editor-File and Code Templates-Files-Classes 在public class上方添加如下注释

        /**
        * @Description: ${DESCRIPTION}
        * @Author: ${USER}
        * @Date: ${DATE} ${TIME}
        */

2. idea支持方法自动注释  
    File-Settings-Editor-Live Templates 选择右方+号， 选择2.Template Group新建group，命名为MyGroup  

    选中新建的MyGroup，再次选中右方+号，选择1.Live Template新建模板 

    Abbreviation中填*， Template text内容如下：  

        **
        * @Description $end$
        * @Author $author$
        * @Date $date$ $time$
        * @Param $param$ 
        * @Return $return$
        */

    选择右方Edit variables按钮为上文中的变量配置对应表达式（Expression）  
    author 对应的Expression为 user()  
    date 对应的Expression为 date()  
    time 对应的Expression为 time()  
    param 对应的Expression为 methodParameters()  
    return 对应的Expression为 methodReturnType()  
    此时下方会有一个警告，No application contexts. Define  
    选中Define按钮，选择Java  
    然后选择ok保存。

