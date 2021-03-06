本文例子基于FreeBSD/Linux实现，windows环境请自己做出相应修改）  
配置管理的一个重要使命是保证数据的安全性，防止服务器应硬盘损坏、误操作造成数据无法恢复的灾难性后果。因此制定一个完整的备份策略非常重要。 

 一般来说，备份策略应规定如下几部分内容：备份频度、备份方式、备份存放地点、备份责任人、灾难恢复检查措施及规定。 

备份频度、存放地点等内容可以根据自己的实际情况自行制定；本文重点描述备份方式。 

svn备份一般采用三种方式：1）svnadmin dump 2)svnadmin hotcopy 3)svnsync. 

注意，svn备份不宜采用普通的文件拷贝方式（除非你备份的时候将库暂停），如copy命令、rsync命令。 
笔者曾经用 rsync命令来做增量和全量备份，在季度备份检查审计中，发现备份出来的库大部分都不可用，因此最好是用svn本身提供的功能来进行备份。 

优缺点分析： 
============== 
第一种svnadmin dump是官方推荐的备份方式，优点是比较灵活，可以全量备份也可以增量备份，并提供了版本恢复机制。   
  缺点是：如果版本比较大，如版本数增长到数万、数十万，那么dump的过程将非常慢；备份耗时，恢复更耗时；不利于快速进行灾难恢复。   
  个人建议在版本数比较小的情况下使用这种备份方式。   
第二种svnadmin hotcopy原设计目的估计不是用来备份的，只能进行全量拷贝，不能进行增量备份；   
   优点是：备份过程较快，灾难恢复也很快；如果备份机上已经搭建了svn服务，甚至不需要恢复，只需要进行简单配置即可切换到备份库上工作。   
   缺点是：比较耗费硬盘，需要有较大的硬盘支持（俺的备份机有1TB空间，呵呵）。   
第三种svnsync实际上是制作2个镜像库，当一个坏了的时候，可以迅速切换到另一个。不过，必须svn1.4版本以上才支持这个功能。   
  优点是：当制作成2个镜像库的时候起到双机实时备份的作用；   
  缺点是：当作为2个镜像库使用时，没办法做到“想完全抛弃今天的修改恢复到昨晚的样子”；而当作为普通备份机制每日备份时，操作又较前2种方法麻烦。        


下面具体描述这三种的备份的方法： 
=============== 

1、svnadmin dump备份工具 
------------------------ 
这是subversion官方推荐的备份方式。 

1）定义备份策略：  
   备份频度：每周六进行一次全量备份，每周日到周五进行增量备份   
   备份地点：备份存储路径到/home/backup/svn/  
   备份命名：全量备份文件名为：weekly_fully_backup.yymmdd,增量备份文件命名为：daily-incremental-backup.yymmdd  
   备份时间：每晚21点开始  
   备份检查：每月末进行svnadmin load恢复试验。  
2）建立全量备份脚本：  
   在~/下建立一个perl脚本文件，名为weekly_backup.pl，执行全量备份，并压缩备份文件，代码如下(本代码只针对一个库的备份，如果是多个库请做相应改动)：   
```
    #!/usr/bin/perl -w 
    my $svn_repos="/home/svn/repos/project1"; 
    my $backup_dir="/home/backup/svn/"; 
    my $next_backup_file = "weekly_fully_backup.".`date +%Y%m%d`; 

    $youngest=`svnlook youngest $svn_repos`; 
    chomp $youngest; 

    print "Backing up to revision $youngestn"; 
    my $svnadmin_cmd="svnadmin dump --revision 0youngest $svn_repos >$backup_dir/$next_backup_file"; 
    `$svnadmin_cmd`; 
    open(LOG,">$backup_dir/last_backed_up"); #记录备份的版本号 
    print LOG $youngest; 
    close LOG; 
    #如果想节约空间，则再执行下面的压缩脚本 
    print "Compressing dump file...n"; 
    print `gzip -g $backup_dir/$next_backup_file`; 
```
3）建立增量备份脚本： 
  在全量备份的基础上，进行增量备份：在~/下建立一个perl脚本文件，名为：daily_backup.pl，代码如下： 
```
    #!/usr/bin/perl -w 
    my $svn_repos="/home/svn/repos/project1"; 
    my $backup_dir="/home/backup/svn/"; 
    my $next_backup_file = "daily_incremental_backup.".`date +%Y%m%d`; 

    open(IN,"$backup_dir/last_backed_up"); 
    $previous_youngest = <IN>; 
    chomp $previous_youngest; 
    close IN; 

    $youngest=`svnlook youngest $svn_repos`; 
    chomp $youngest; 
    if ($youngest eq $previous_youngest) 
    { 
      print "No new revisions to backup.n"; 
      exit 0; 
    } 
    my $first_rev = $previous_youngest + 1; 
    print "Backing up revisions $youngest ...n"; 
    my $svnadmin_cmd = "svnadmin dump --incremental --revision $first_revyoungest $svn_repos > $backup_dir/$next_backup_file"; 
    `$svnadmin_cmd`; 
    open(LOG,">$backup_dir/last_backed_up"); #记录备份的版本号 
    print LOG $youngest; 
    close LOG; 
    #如果想节约空间，则再执行下面的压缩脚本 
    print "Compressing dump file...n"; 
    print `gzip -g $backup_dir/$next_backup_file`; 
```
4）配置/etc/crontab文件 
配置 /etc/crontab 文件，指定每周六执行weekly_backup.pl，指定周一到周五执行daily_backup.pl; 
具体步骤俺就不啰嗦了. 

5）备份恢复检查 
在月底恢复检查中或者在灾难来临时，请按照如下步骤进行恢复：恢复顺序从低版本逐个恢复到高版本；即，先恢复最近的一次完整备份 weekly_full_backup.071201（举例），然后恢复紧挨着这个文件的增量备份 daily_incremental_backup.071202，再恢复后一天的备份071203，依次类推。如下： 
```
user1>mkdir newrepos 
user1>svnadmin create newrepos 
user1>svnadmin load newrepos < weekly_full_backup.071201 
user1>svnadmin load newrepos < daily_incremental_backup.071202 
user1>svnadmin load newrepos < daily_incremental_backup.071203 
.... 
```
如果备份时采用了gzip进行压缩，恢复时可将解压缩和恢复命令合并，简单写成： 
```
user1>zcat weekly_full_backup.071201 | svnadmin load newrepos 
user1>zcat daily_incremental_backup.071202 | svnadmin load newrepos 
...
```
(这部分内容很多参考了《版本控制之道》)  

2、svnadmin hotcopy整库拷贝方式 
------------------------- 
svnadmin hotcopy是将整个库都“热”拷贝一份出来，包括库的钩子脚本、配置文件等；任何时候运行这个脚本都得到一个版本库的安全拷贝，不管是否有其他进程正在使用版本库。 
因此这是俺青睐的备份方式。 

1）定义备份策略  
  备份频度：每天进行一次全量备份， 
  备份地点：备份目录以日期命名，备份路径到 /home/backup/svn/${mmdd} 
  备份保留时期：保留10天到15天，超过15天的进行删除。 
  备份时间：每晚21点开始 
  备份检查：备份完毕后自动运行检查脚本、自动发送报告。 

2）建立备份脚本  
在自己home目录 ~/下创建一个文件，backup.sh： 
```
#!/bin/bash 
SRCPATH=/home/svn/repos/; #定义仓库parent路径 
DISTPATH=/home/backup/svn/`date +%m%d`/ ; #定义存放路径; 
if [ -d "$DISTPATH" ] 
then 
else 
   mkdir $DISTPATH 
   chmod g+s $DISTPATH 
fi 
echo $DISTPATH 
svnadmin hotcopy $SRCPATH/Project1 $DISTPATH/Project1 >/home/backup/svn/cpreport.log 2>&1; 
svnadmin hotcopy $SRCPATH/Project2 $DISTPATH/Project2 
cp $SRCPATH/access  $DISTPATH; #备份access文件 
cp $SRCPATH/passwd  $DISTPATH; #备份passwd文件 
perl /home/backup/svn/backup_check.pl #运行检查脚本 
perl /home/backup/svn/deletDir.pl  #运行删除脚本，对过期备份进行删除。 
```
3）建立检查脚本 
在上面指定的地方/home/backup/svn/下建立一个perl脚本：backup_check.pl  
备份完整性检查的思路是：对备份的库运行 svnlook youngest，如果能正确打印出最新的版本号，则表明备份文件没有缺失；如果运行报错，则说明备份不完整。我试过如果备份中断，则运行svnlook youngest会出错。   
perl脚本代码如下： 
```
#! /usr/bin/perl 
## Author:xuejiang 
## 2007-11-10 
## http://www.scmbbs.com 
use strict; 
use Carp; 
use Net::SMTP; 

#### defined the var ####### 

my $smtp =Net::SMTP->new('mail.scmbbs.com', Timeout => 30, Debug => 0)|| die "cann't connect to mail.scmbbs.comn"; 

my $bkrepos="/home/backup/svn/".&get_day;#定义备份路径 
my $ssrepos="http://www.scmbbs.com/repos";#定义仓库url 
my @repos = ("project1","project2"); 

my $title="echo "如下是昨晚备份结果与真实库对比的情况，如果给出备份版本数，则表示备份成功；如果给报错信息或没有备份版本数，则表示备份失败：" >./report"; 
system $title  || die "exec failedn"; 
foreach my $myrepos(@repos) 
{ 
    my $bkrepos1=$bkrepos."/".$myrepos; 
  my $ssrepos1=$ssrepos."/".$myrepos; 
  my $svnlookbk1 = "echo "$myrepos 昨晚备份的版本是：">>./report;svnlook youngest ".$bkrepos1." >> ./report 2>&1"; 
  my $svnlookss1 = "echo "$myrepos 真实库中的最新版本及最后修改时间是：">>./report;svn log -r'HEAD' ".$ssrepos1." >> ./report 2>&1"; 
  system $svnlookbk1 || die "exec failedn"; 
  system $svnlookss1 || die "exec failedn"; 

} 

my $body       ="echo "=========================================================================" >>./report"; 
my $bottom     ="echo "备份位置：来自http://www.scmbbs.com的".$bkrepos."" >>./report"; 

system $body       || die "exec failedn"; 
system $bottom     || die "exec failedn"; 


###### report the result #### 


open(SESAME,"./report")|| die "can not open ./report"; 
my @svnnews = <SESAME>; 
close(SESAME); 
foreach my $line1 (@svnnews) 
{ 
      print $line1."n"; 
} 

my @email_addresses =("scm@list.scmbbs.com","leader1@scmbbs.com","leader2@scmbbs.com"); 
my $to              = join(', ', @email_addresses); 
$smtp->mail("scm@scmbbs.com"); 
$smtp->recipient(@email_addresses); 
$smtp->data(); 
$smtp->datasend("Toton"); 
$smtp->datasend("From: svnReport@scmbbs.comn"); 
$smtp->datasend("Subject:svn备份检查报告".&get_today."n"); 
$smtp->datasend("Reply-to:scm@scmbbs.comn"); 
$smtp->datasend("@svnnews"); 
$smtp->dataend(); 
$smtp->quit; 


############# 


sub get_today 
{ 
my( $sec, $min, $hour, $day, $month, $year ) = localtime( time() ); 
$year += 1900; 
$month++; 
my $today = sprintf( "%04d%02d%02d", $year, $month, $day); 
return $today; 
} 
sub get_day 
{ 
    my( $sec, $min, $hour, $day, $month, $year ) = localtime( time() ); 
$year += 1900; 
$month++; 
my $today = sprintf( "%02d%02d", $month, $day); 
return $today; 
} 
```
   
4)定义删除脚本  
  由于是全量备份，所以备份不宜保留太多，只需要保留最近10来天的即可，对于超过15天历史的备份基本可以删除了。 
  在/home/backup/svn/下建立一个perl脚本：deletDir.pl 
  (注意，删除svn备份库可不像删除普通文件那么简单） 
  脚本代码请参看我的另一个帖子：http://www.scmbbs.com/cn/systp/2007/12/systp6.php 

5）修改/etc/crontab 文件  
  在该文件中指定每晚21点执行“backup.sh”脚本。 

3、svnsync备份 
----------------------- 
参阅：http://www.scmbbs.com/cn/svntp/2007/11/svntp4.php 
使用svnsync备份很简单，步骤如下： 
  1）在备份机上创建一个空库：svnadmin create Project1 
  2）更改该库的钩子脚本pre-revprop-change（因为svnsync要改这个库的属性，也就是要将源库的属性备份到这个库，所以要启用这个脚本）:   
    cd SMP/hooks; 
    cp pre-revprop-change.tmpl pre-revprop-change; 
    chmod 755 pre-revprop-change; 
    vi pre-revprop-change; 
    将该脚本后面的三句注释掉，或者干脆将它弄成一个空文件。 
  3）初始化，此时还没有备份任何数据： 
  svnsync init file:///home/backup/svn/svnsync/Project1/  http://svntest.subversion.com/repos/Project1 
    语法是：svnsync init {你刚创建的库url} {源库url} 
    注意本地url是三个斜杠的：/// 
  4）开始备份（同步）： 
    svnsync sync file:///home/backup/svn/svnsync/Project1 
  5）建立同步脚本 
    备份完毕后，建立钩子脚本进行同步。在源库/hooks/下建立/修改post-commit脚本，在其中增加一行，内容如下： 
```
/usr/bin/svnsync sync  --non-interactive file:///home/backup/svn/svnsync/Project1 
```

你可能已经注意到上面的备份似乎都是本地备份，不是异地备份。实际上，我是通过将远程的备份机mount（请参阅mount命令）到svn服务器上来实现的，逻辑上看起来是本地备份，物理上实际是异地备份。
  
  参考：  
  [SVN服务器几种备份策略----------重点svnsync备份](http://www.blogjava.net/jasmine214--love/archive/2010/09/28/333223.html)
