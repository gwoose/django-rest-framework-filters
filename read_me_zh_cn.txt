djangorestframework-filters 使用
djangorestframework-filters 是 Django REST framework和Django filter的扩展,通过它可以方便跨关系过滤 。
由于合并到到django-filter，历史上扩展的一些特性和修复缩减了。

1. 特性
× 易于跨模型关系过滤
× 支持跨模型关系方法过滤
× 自动协商过滤不等(param!=value)
* 后端缓存提高了效率

2. 安装
pip install djangorestframework-filters

3. 使用
从感官上可以感觉到django-filter 到django-rest-framework-filters升级
× 导入rest_framework_filters，不再导入django_filters
× 使用rest_framework_filters后端替代django_filter提供的后端

# django-filter
from django_filters.rest_framework import FilterSet, filters

class ProductFilter(FilterSet):
    manufacturer = filters.ModelChoiceFilter(queryset=Manufacturer.objects.all())
    ...


# django-rest-framework-filters
import rest_framework_filters as filters

class ProductFilter(filters.FilterSet):
    manufacturer = filters.ModelChoiceFilter(queryset=Manufacturer.objects.all())
    ...
为了使用django-rest-framework 后端，添加下面的内容到settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': (
        'rest_framework_filters.backends.DjangoFilterBackend', ...
一旦上面的配置完成，你就可以继续使用所有django-filter中的过滤器了。

4. 跨关系过滤
使用RelatedFilter做跨关系过滤
from rest_framework import viewsets
import rest_framework_filters as filters


class ManagerFilter(filters.FilterSet):
    class Meta:
        model = Manager
        fields = {'name': ['exact', 'in', 'startswith']}


class DepartmentFilter(filters.FilterSet):
    manager = filters.RelatedFilter(ManagerFilter, name='manager', queryset=Manager.objects.all())

    class Meta:
        model = Department
        fields = {'name': ['exact', 'in', 'startswith']}


class CompanyFilter(filters.FilterSet):
    department = filters.RelatedFilter(DepartmentFilter, name='department', queryset=Department.objects.all())

    class Meta:
        model = Company
        fields = {'name': ['exact', 'in', 'startswith']}


# company viewset
class CompanyView(viewsets.ModelViewSet):
    filter_class = CompanyFilter
    ...

调用实例：
/api/companies?department__name=Accounting
/api/companies?department__manager__name__startswith=Bob

5. 可调用（函数）queryset 
因为RealtedFilter是ModelChoiceFilter的子类，所以queryset参数可以是可调用函数。下面的例子中department集合就是登录用户所在公司的部门集合
def departments(request):
    company = request.user.company
    return company.department_set.all()

class EmployeeFilter(filters.FilterSet):
    department = filters.RelatedFilter(filterset=DepartmentFilter, queryset=departments)

6. 递归关系
递归过滤也是允许的。但是只能指定全模块路径，如：
class PersonFilter(filters.FilterSet):
    name = filters.AllLookupsFilter(name='name')
    best_friend = filters.RelatedFilter('people.views.PersonFilter', name='best_friend', queryset=Person.objects.all())

    class Meta:
        model = Person

7. 支持Filter.method
django_filters.MethodFilter 已不建议使用，它已经重载为所有filter 类的method参数。它结合了老的rest_framework_filters.MethodFilter,但减少了模板，更简单使用。
 * 不必再做控制检测
 * 你可以使用任何过滤类 （如CharFilter,BooleanFilter等等）并为你校验输入；
 × 参数的列表由原来的(name, qs, value) 换为(qs, name, value)

class PostFilter(filters.FilterSet):
    # Note the use of BooleanFilter, the original model field's name, and the method argument.
    is_published = filters.BooleanFilter(name='date_published', method='filter_is_published')

    class Meta:
        model = Post
        fields = ['title', 'content']

    def filter_is_published(self, qs, name, value):
        """
        `is_published` is based on the `date_published` model field.
        If the publishing date is null, then the post is not published.
        """
        # incoming value is normalized as a boolean by BooleanFilter
        isnull = not value
        lookup_expr = LOOKUP_SEP.join([name, 'isnull'])

        return qs.filter(**{lookup_expr: isnull})

class AuthorFilter(filters.FilterSet):
    posts = filters.RelatedFilter('PostFilter', queryset=Post.objects.all())

    class Meta:
        model = Author
        fields = ['name']

上面的定义支持一下过滤调用：
/api/posts?is_published=true
/api/authors?posts__is_published=true
第一个API调用中，filter method接受一个posts的查询集，第二个调用中，它接受一个users的查询集。
filter method在在跨模型关系修改查询名称，允许你查找published posts或这published posts的用户；

8. 自动化过滤器 反向/排除
过滤器组支持使用param!=value符号自动排除，内部实现是在过滤器上设置exclude属性
/api/page?title!=the%20Parh
这个符号支持常规的和排除过滤。比如下面搜索所有articles中包含Hello同步排除包含World的article
/api/articles?tilte__contains=Hello&title__contains!=World
注意绝大多数的过滤器只包含单个查询参数。上面的例子中title__contains和title_contains!被解析为两个独立查询参数。下面的实例可能是无效的，尽管它依赖于单独过滤器类设计
/api/articles?title__contains=Hello&title__contains!=World&title_contains!=Friend

8. 字段上允许任何查找类型
如果你需要在一个字段上做多种查找，django-filter为Meta.fields提供了字典符号.
class ProductFilter(filters.FilterSet):
    class Meta:
        model = Product
        fields = {
            'price': ['exact', 'lt', 'gt', ...],
        }
django-rest-framework-filters 允许你在任何字段上启动所有可能的查找。这可通过AllLookupsFilter或者使用在Meta.fields字典符号中"__all__"
但它们生成的过滤器不会重写你声明的过滤器。
注意使用所有查找的时，这会允许用户构造请求，可能有泄露数据的危险。所以负责任的使用这个功能。
class ProductFilter(filters.FilterSet):
    # Not overridden by `__all__`
    price__gt = filters.NumberFilter(name='price', lookup_expr='gt', label='Minimum price')

    class Meta:
        model = Product
        fields = {
            'price': '__all__',
        }

# or
class ProductFilter(filters.FilterSet):
    price = filters.AllLookupsFilter()

    # Not overridden by `AllLookupsFilter`
    price__gt = filters.NumberFilter(name='price', lookup_expr='gt', label='Minimum price')

    class Meta:
        model = Product

9. 混合jdango-filter和django-rest-framework-filters
可以混合。django-rest-framework-filter是django-filter的简单扩展。注意RelatedFilter和其他django-rest-framework-filters特性被设计为和django-filter配合的且不会作用于django_filters.FilterSet。但是RelatedFilter.filterset目标是指向django_filters.FilterSet。无论是从包上还是FilterSet继承都是和其他DRF backend兼容的。
# valid
class VanillaFilter(django_filters.FilterSet):
    ...

class DRFFilter(rest_framework_filters.FilterSet):
    vanilla = rest_framework_filters.RelatedFilter(filterset=VanillaFilter, queryset=...)


# invalid
class DRFFilter(rest_framework_filters.FilterSet):
    ...

class VanillaFilter(django_filters.FilterSet):
    drf = rest_framework_filters.RelatedFilter(filterset=DRFFilter, queryset=...)

10. 警告和限制
MutiWiget 是不兼容的
djangorestframework-filters 不兼容与form 组件。这些组件转化的查询名称和过滤器的属性名称不同。自定义组件通常也会受到限制，有影响过滤器包括RangeFilter,DateTimeFromToRangeFilter,DateFormRangeFilter,TimeRangeFilter,NumbricRangeFilter
为了演示这种不兼容，查看下满的filterset
class PostFilter(FilterSet):
   publish_date = filters.DateFormToRangeFilter()
上述筛选器允许用户对发布日期执行范围查询。过滤器类在内部使用MultiWidget分别解析上界和下界值。不兼容性在于MultiWidget将一个索引附加到它的内部部件名称。它不解析发布日期，而是期望发布日期为0和发布日期为1。可以通过在querystring中包含属性名来解决这个问题，但不建议这样做。

?publish_date_0=2016-01-01&publish_date_1=2016-02-01&publish_date=

不鼓励使用MultiWidget，因为：
* 出于类似原因，core-api字段自省失败
* 与_min和_max相比，_0和_1对API的友好程度较低
推荐的解决方案是：
* 为每个子小部件创建单独的过滤器（例如publish_date_min和publish_date_max）。
* 使用基于CSV的过滤器，例如从BaseCSVFilter / BaseInFilter / BaseRangeFilter派生的过滤器。 例如，
?publish_date__range=2016-01-01,2016-02-01

11. 迁移到 to 1.0版本
× RelatedFilter.queryset是必需的。
相关的过滤器集模型不再用于为RelatedFilter.queryset提供默认值。此更改减少了在呈现的筛选器表单中无意暴露数据的可能性。现在必须显式地提供queryset参数，或覆盖get queryset()方法(请参阅queryset callables)。

× get_filters() 重命名为 expand_filters()
django-filter在它的API中添加了一个get_filters()类方法，因此这个方法已经被重命名。
