# 是什么
    Celery 是 Python 语言实现的分布式队列服务，除了支持即时任务，还支持定时任务
# 角色
- Task
    任务(Task)就是你要做的事情，例如一个注册流程里面有很多任务，给用户发验证邮件就是一个任务，这种耗时任务可以交给Celery去处理，还有一种任务是定时任务，比如每天定时统计网站的注册人数，这个也可以交给Celery周期性的处理。
- Broker
    Broker 的中文意思是经纪人，指为市场上买卖双方提供中介服务的人。在Celery中它介于生产者和消费者之间经纪人，这个角色相当于数据结构中的队列。例如一个Web系统中，生产者是处理核心业务的Web程序，业务中可能会产生一些耗时的任务，比如短信，生产者会将任务发送给 Broker，就是把这个任务暂时放到队列中，等待消费者来处理。消费者是 Worker，是专门用于执行任务的后台服务。Worker 将实时监控队列中是否有新的任务，如果有就拿出来进行处理。Celery 本身不提供队列服务，一般用 Redis 或者 RabbitMQ 来扮演 Broker 的角色
- Worker
    Worker 就是那个一直在后台执行任务的人，也称为任务的消费者，它会实时地监控队列中有没有任务，如果有就立即取出来执行。
- Beat
    Beat 是一个定时任务调度器，它会根据配置定时将任务发送给 Broker，等待 Worker 来消费。
- Backend
    Backend 用于保存任务的执行结果，每个任务都有返回值，比如发送邮件的服务会告诉我们有没有发送成功，这个结果就是存在Backend中，当然我们并不总是要关心任务的执行结果。


# 创建任务方法
- apply_async(self, args=None, kwargs=None, task_id=None, producer=None, link=None, link_error=None, shadow=None, **options)
    task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
    指定添加到队列
    add.apply_async(queue='priority.high')
    指定读取队列
    celery worker -l info -Q celery,priority.high

- delay(*args, **kwargs)
    task.delay(arg1, arg2, kwarg1='x', kwarg2='y')

- signature
   signature('tasks.add', args=(2, 2), countdown=10)
   add.subtask((2, 2), countdown=10)
   add.s(2, 2, debug=True)
   add.s(2, 2).set(countdown=1)
   execute:
     add.s(2, 2).delay()
     add.subtask(args, kwargs, **options).apply_async()
     add.s(2, 2)() 本进程内执行


# worker执行
- start
    celery worker --help 具体选项详情
    celery worker -A proj -l info -Q hipri,lopri
- stop
   ps auxww | grep 'celery worker' | awk '{print $2}' | xargs kill -9
- restart
   kill -HUP $pid

- 周期任务celery beat
  同时只能有一个触发器，否则会有任务重复
  celery beat --help
  celery worker -B


# 保存结果
app = Celery('tasks', backend='amqp', broker='amqp://')
app = Celery('tasks', backend='redis://localhost', broker='amqp://')

redis backend key prefix task_keyprefix = 'celery-task-meta-'
meta = {
            'status': state,
            'result': result,
            'traceback': traceback,
            'children': self.current_task_children(request),
            'task_id': bytes_to_str(task_id),
            'date_done': date_done,
        }



# 配置文件
BROKER_URL = 'amqp://'
CELERY_RESULT_BACKEND = 'amqp://'
CELERY_RESULT_BACKEND = 'redis://localhost/0'

CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT=['json']
CELERY_TIMEZONE = 'Europe/Oslo'
CELERY_ENABLE_UTC = True
CELERY_ROUTES = {
    'tasks.add': 'low-priority',
}

CELERY_ANNOTATIONS = {
    'tasks.add': {'rate_limit': '10/m'}
}

# 队列路由
默认的队列名字是celery
改变默认队列设置：
from kombu import Exchange, Queue

routing_key 消息投递时的key

bind_key队列绑定到exchange的key


CELERY_DEFAULT_QUEUE = 'default'
CELERY_QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
)
queue.bind(queue_name, exchange_name, routing_key): Binds a queue to an exchange with a routing key


CELERY_ROUTES = {'feed.tasks.import_feed': {'queue': 'feeds'}}

class MyRouter(object):
    # task是任务的名字
    def route_for_task(self, task, args=None, kwargs=None):
        if task == 'myapp.tasks.compress_video':
            return {'exchange': 'video',
                    'exchange_type': 'topic',
                    'routing_key': 'video.compress'}
        return None

#--配置task的路由
app.conf.task_routes = {
    'tasks.login.execute_login_task': {
        'queue': 'login',
        'serializer': 'json',
        'routing_key': 'login_all'
    },
    'tasks.login.do_login': {
        'queue': 'login',
        'serializer': 'json',
        'routing_key': 'login_each'
    },
    'tasks.login.do_gen_cookie': {
        'queue': 'gen_cookies',
        'serializer': 'json',
        'routing_key': 'gen_cookies'
    },
    'tasks.user.*': {
        'queue': 'user',
        'routing_key': 'user.all',
        'serializer': 'json'
    },
}

task_routes 中在有routing_key的情况下queue有作用？

设置目标的顺序：
1、使用routers的定义
2、使用Task.apply_async()的路由参数
3、Task自身定义的路由


# 
The non-AMQP backends like ghettoq does not support exchanges, 
so they require the exchange to have the same name as the queue





# combu
You may deliver to a queue directly, bypassing the brokers routing mechanisms, 
by using the “anon-exchange”: set the exchange parameter to the empty string, 
and set the routing key to be the name of the queue:

>>> producer.publish(
...     {'hello': 'world'},
...     exchange='',
...     routing_key=task_queue.name,
... )


# convert to anon-exchange, when exchange not set and direct ex.

# 没有设置exchange 或者 routing_key 队列名字作为routing_key
if (not exchange or not routing_key) and exchange_type == 'direct':
    exchange, routing_key = '', qname

celery/app/amqp.py
_create_task_sender -> send_task_message -> publish