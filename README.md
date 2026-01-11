# django_sprint4
# Макаренков Андрей, МО02
## Отчет
## 1. Общая архитектура проекта и организация приложений

Проект реализован с использованием фреймворка Django и построен по классической MVC-архитектуре (Model–Template–View). Основная бизнес-логика вынесена в приложение `blog`, а вспомогательные компоненты — в приложение `core`.

Структура проекта обеспечивает:

* разделение ответственности;
* повторное использование кода;
* удобство поддержки и масштабирования.

Пример структуры проекта:

```text
django_sprint4/
├── blogicum/
│   ├── blog/
│   │   ├── admin.py
│   │   ├── forms.py
│   │   ├── models.py
│   │   ├── urls.py
│   │   ├── views.py
│   │   └── migrations/
│   ├── core/
│   │   ├── mixins.py
│   │   └── utils.py
│   ├── templates/
│   ├── static/
│   ├── settings.py
│   └── manage.py
```

Приложение `blog` отвечает за:

* публикации;
* комментарии;
* категории;
* пользовательские профили.

Приложение `core` используется для вынесения общей логики, применяемой сразу в нескольких представлениях.

---

## 2. Модели данных и связи между ними

### Модель категории

Категории используются для логической группировки постов и управления их отображением на сайте.

```python
class Category(models.Model):
    title = models.CharField(max_length=256)
    slug = models.SlugField(unique=True)
    description = models.TextField()
    is_published = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

Модель содержит флаг публикации `is_published`, который позволяет скрывать категории без их удаления из базы данных.

---

### Модель публикации (Post)

Пост является центральной сущностью проекта и связан с пользователем и категорией.

```python
class Post(models.Model):
    title = models.CharField(max_length=256)
    text = models.TextField()
    pub_date = models.DateTimeField()
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name="posts"
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="posts"
    )
    image = models.ImageField(
        upload_to="posts/",
        null=True,
        blank=True
    )
    is_published = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

Связи модели:

* при удалении автора его посты удаляются (`CASCADE`);
* при удалении категории пост сохраняется, но категория становится пустой (`SET_NULL`).

---

### Модель комментария

Комментарии связаны с конкретным постом и пользователем.

```python
class Comment(models.Model):
    text = models.TextField()
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name="comments"
    )
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name="comments"
    )
    created_at = models.DateTimeField(auto_now_add=True)
```

Такое решение позволяет автоматически удалять комментарии при удалении поста.

---

## 3. Настройка административной панели Django

Для управления контентом проекта используется стандартная административная панель Django. Все основные модели зарегистрированы с дополнительными настройками отображения.

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = (
        "title",
        "author",
        "category",
        "pub_date",
        "is_published"
    )
    list_filter = ("is_published", "category", "author")
    search_fields = ("title", "text")
    date_hierarchy = "pub_date"
```

```python
@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ("title", "slug", "is_published")
    list_editable = ("is_published",)
    search_fields = ("title",)
```

```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ("author", "post", "created_at")
```

Административная панель используется для:

* управления публикациями;
* модерации комментариев;
* скрытия и отображения контента.

---

## 4. Маршрутизация и работа с URL

Маршруты приложения описаны в файле `blog/urls.py`. Используются осмысленные имена URL-параметров.

```python
app_name = "blog"

urlpatterns = [
    path("", views.MainPostListView.as_view(), name="index"),
    path(
        "category/<slug:category_slug>/",
        views.CategoryPostListView.as_view(),
        name="category_posts",
    ),
    path(
        "profile/<slug:username>/",
        views.UserPostsListView.as_view(),
        name="profile",
    ),
    path(
        "posts/<int:pk>/",
        views.PostDetailView.as_view(),
        name="post_detail",
    ),
    path(
        "posts/create/",
        views.PostCreateView.as_view(),
        name="create_post",
    ),
    path(
        "posts/<int:pk>/edit/",
        views.PostUpdateView.as_view(),
        name="edit_post",
    ),
    path(
        "posts/<int:pk>/delete/",
        views.PostDeleteView.as_view(),
        name="delete_post",
    ),
]
```

Использование именованных маршрутов позволяет удобно выполнять перенаправления через `reverse()` и `redirect()`.

---

## 5. Представления: отображение списков постов

### Главная страница сайта

```python
class MainPostListView(ListView):
    model = Post
    template_name = "blog/index.html"
    paginate_by = 10

    def get_queryset(self):
        return get_posts_metadata()
```

На главной странице отображаются только опубликованные посты, прошедшие фильтрацию по дате публикации и статусу категории.

---

### Страница категории

```python
class CategoryPostListView(ListView):
    template_name = "blog/category.html"
    paginate_by = 10

    def get_queryset(self):
        self.category = get_object_or_404(
            Category,
            slug=self.kwargs["category_slug"],
            is_published=True
        )
        return get_posts_metadata(self.category.posts.all())
```

Пользователь видит только те посты, которые принадлежат опубликованной категории.

---

### Профиль пользователя

```python
class UserPostsListView(ListView):
    template_name = "blog/profile.html"
    paginate_by = 10

    def get_queryset(self):
        self.author = get_object_or_404(
            User,
            username=self.kwargs["username"]
        )
        is_author = self.author == self.request.user
        posts = self.author.posts.all()
        return get_posts_metadata(posts, is_author=is_author)
```

Если страницу профиля открывает владелец, ему отображаются все его посты, включая черновики. Для остальных пользователей показываются только опубликованные записи.

## 6. Детальный просмотр публикации и контроль доступа

Для отображения отдельного поста используется CBV `DetailView`.
В представлении реализована дополнительная логика проверки прав доступа: неопубликованные посты доступны только их авторам.

```python
class PostDetailView(DetailView):
    model = Post
    template_name = "blog/detail.html"

    def get_object(self):
        post = get_object_or_404(
            Post.objects.select_related("category", "author"),
            pk=self.kwargs["pk"]
        )

        if self.request.user != post.author:
            post = get_object_or_404(
                Post,
                pk=post.pk,
                is_published=True,
                category__is_published=True,
                pub_date__lte=timezone.now()
            )
        return post
```

Таким образом:

* автор видит свой пост в любом состоянии;
* остальные пользователи — только опубликованные записи;
* исключается возможность просмотра скрытого контента.

---

## 7. Формы и работа с пользовательским вводом

Для взаимодействия с пользователем в проекте используются формы Django, основанные на `ModelForm`.

### Форма редактирования профиля

```python
class UserEditForm(forms.ModelForm):
    class Meta:
        model = User
        fields = (
            "first_name",
            "last_name",
            "email",
        )
```

Форма позволяет пользователю редактировать только основные данные профиля, не затрагивая служебные поля.

---

### Форма создания и редактирования поста

```python
class PostEditForm(forms.ModelForm):
    class Meta:
        model = Post
        exclude = ("author", "created_at")
        widgets = {
            "pub_date": forms.DateTimeInput(
                attrs={"type": "datetime-local"},
                format="%Y-%m-%dT%H:%M"
            )
        }
```

Использование `exclude` упрощает поддержку формы и исключает ручную установку автора поста.

---

### Форма комментария

```python
class CommentEditForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ("text",)
```

Минимальный набор полей снижает вероятность ошибок ввода.

---

## 8. Создание, обновление и удаление постов

### Создание публикации

```python
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostEditForm
    template_name = "blog/create.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)

    def get_success_url(self):
        return reverse(
            "blog:profile",
            kwargs={"username": self.request.user.username}
        )
```

Создание постов доступно только авторизованным пользователям.
Автор назначается автоматически.

---

### Обновление публикации

```python
class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostEditForm
    template_name = "blog/create.html"

    def dispatch(self, request, *args, **kwargs):
        if self.get_object().author != request.user:
            return redirect("blog:post_detail", pk=self.kwargs["pk"])
        return super().dispatch(request, *args, **kwargs)
```

Редактирование возможно только для владельца поста.

---

### Удаление публикации

```python
class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    template_name = "blog/create.html"

    def dispatch(self, request, *args, **kwargs):
        if self.get_object().author != request.user:
            return redirect("blog:post_detail", pk=self.kwargs["pk"])
        return super().dispatch(request, *args, **kwargs)
```

Проверка прав реализована единообразно для всех операций с постом.

---

## 9. Работа с комментариями пользователей

### Добавление комментария

```python
class CommentCreateView(LoginRequiredMixin, CreateView):
    model = Comment
    form_class = CommentEditForm
    template_name = "blog/comment.html"

    def dispatch(self, request, *args, **kwargs):
        self.post = get_post_data(self.kwargs)
        return super().dispatch(request, *args, **kwargs)

    def form_valid(self, form):
        form.instance.author = self.request.user
        form.instance.post = self.post
        if self.post.author != self.request.user:
            self.send_author_email()
        return super().form_valid(form)
```

Комментарий автоматически связывается:

* с текущим пользователем;
* с постом, определённым по URL.

---

## 10. Использование миксинов для комментариев

Для исключения дублирования логики проверки прав был реализован отдельный миксин.

```python
class CommentMixinView(LoginRequiredMixin, View):
    model = Comment
    template_name = "blog/comment.html"
    pk_url_kwarg = "comment_pk"

    def dispatch(self, request, *args, **kwargs):
        if self.get_object().author != request.user:
            return redirect(
                "blog:post_detail",
                pk=self.kwargs["pk"]
            )
        get_post_data(self.kwargs)
        return super().dispatch(request, *args, **kwargs)

    def get_success_url(self):
        return reverse(
            "blog:post_detail",
            kwargs={"pk": self.kwargs["pk"]}
        )
```

Миксин используется как для редактирования, так и для удаления комментариев.

---

## 11. Вспомогательные функции и бизнес-логика

### Фильтрация и аннотация постов

```python
def get_posts_metadata(queryset=None, is_author=False):
    if queryset is None:
        queryset = Post.objects.all()

    if not is_author:
        queryset = queryset.filter(
            is_published=True,
            category__is_published=True,
            pub_date__lte=timezone.now()
        )

    return queryset.annotate(
        comment_count=Count("comments")
    ).order_by("-pub_date")
```

Функция используется во всех списках постов и обеспечивает:

* фильтрацию по статусу публикации;
* подсчёт количества комментариев;
* корректную сортировку.

---

### Получение поста из URL

```python
def get_post_data(kwargs):
    return get_object_or_404(
        Post,
        pk=kwargs.get("pk")
    )
```

Используется при работе с комментариями.

---

## Заключение

В рамках проекта реализовано полноценное веб-приложение для публикации и обсуждения контента.
Использованы:

* Django ORM;
* CBV;
* система шаблонов;
* механизмы аутентификации и авторизации;
* миксины и вспомогательные функции.


