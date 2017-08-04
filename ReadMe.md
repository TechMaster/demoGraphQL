# Hướng dẫn thực hành GraphQL và PostgraphQL

## Ví dụ dịch vụ REST truy vấn dữ liệu điển hình
Một status trên Facebook sẽ gồm có:
- text
- nhiều ảnh, video
- danh sách người like, share, react. Với mỗi người like cần có ảnh.
- danh sách các comment

**Câu hỏi:** 
1. Gọi một lệnh REST GET là đủ hay cần phải nhiều lệnh?
2. Nếu gọi nhiều lệnh điều gì sẽ xảy ra?
3. Nếu gọi một lệnh duy nhất nhưng muốn nhận về tập dữ liệu đầy đủ thì trên server sẽ phải lập trình như thế nào?


## Ví dụ Author <--> Post

## Sử dụng JSON để gom nhóm dữ liệu liên quan

```sql
SELECT author FROM author

SELECT row_to_json(author) FROM author

SELECT ROW_TO_JSON(a) FROM author AS a JOIN author_post AS ap ON a.id = ap.author_id WHERE a.id = 10
```

Liệt kê danh sách tác giả và tác phẩm
```sql
SELECT * 
FROM author AS a 
INNER JOIN author_post AS ap ON a.id = ap.author_id
INNER JOIN post AS p ON p.id = ap.post_id ORDER BY a.id
```

Đoạn lệnh này lấy ra tác giả và danh sách các post title
```sql
SELECT a.first_name, a.last_name,
  (ARRAY(SELECT p.title FROM author_post AS ap JOIN post AS p ON ap.post_id = p.id WHERE ap.author_id = a.id)) AS post_id
FROM author AS a
```

Sử dụng json_build_object
```sql
SELECT row_to_json(a) AS author_obj,
  (array(
      SELECT json_build_object('title', p.title, 'id', p.id) 
      FROM author_post AS ap 
      INNER JOIN post AS p ON ap.post_id = p.id 
      WHERE ap.author_id = a.id)
  ) AS posts_by_author
FROM author AS a
WHERE a.id IN (1, 2, 3, 4)
```

Sử dụng hoàn toàn row_to_json
```sql
SELECT row_to_json(a) AS author_obj,
  (array(
      SELECT row_to_json(p) 
      FROM author_post AS ap 
      INNER JOIN post AS p ON ap.post_id = p.id 
      WHERE ap.author_id = a.id)
  ) AS post_by_author
FROM author AS a WHERE a.id in (1, 2, 3, 4)
```

Tham khảo thêm [Hỏi đáp StackOverflow về Postgresql JSON](https://stackoverflow.com/questions/13227142/postgresql-9-2-row-to-json-with-nested-joins)

## Cài đặt
1. Cài đặt Postgresql database bằng Docker
```docker run --name db -p 5432:5432 -e POSTGRES_PASSWORD=abc -d postgres:alpine```
2. Cài đặt Postgraphql
```
npm install -g postgraphql
postgraphql -c postgres://postgres:abc@localhost:5432/shop --watch
```
3. Bổ xung các bảng
```
psql -h 127.0.0.1 -W -U postgres

CREATE TABLE author (
    id SERIAL PRIMARY KEY,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT NOT NULL
)

CREATE TABLE post (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL
)

CREATE TABLE author_post (
    author_id INT REFERENCES author(id),
    post_id INT REFERENCES post(id),
    CONSTRAINT composite_unique_keys UNIQUE(author_id, post_id)
)
```
Sử dụng Mockaroo để sinh dữ liệu giả.

### Truy vấn một post
```
{
  postById(id: 5) {
    nodeId
    id
    title
    content
  }
}
```
### Post cùng với Author
```
{
  allPosts {
    nodes {
      id
      title
      content
      authorPostsByPostId {
        totalCount
        nodes {
          authorByAuthorId {
            firstName
            lastName
            email
          }
        }
      }
    }
  }
}
```
### Kỹ thuật phân trang


### Bổ xung function trong Postgresql để mở rộng chức năng
Hàm tìm kiếm các post
```sql
CREATE FUNCTION search_posts(search TEXT) RETURNS SETOF post AS $$
  SELECT *
  FROM post
  WHERE title ilike ('%' || search || '%') OR content ilike ('%' || search || '%')
$$ language sql stable;

comment on function search_posts(text) is 'Returns posts containing a given search term.';
```

### Liệt kê post theo danh sách ID cho sẵn
Cần truy vấn các post có id trong mảng nhập vào
```sql
CREATE OR REPLACE FUNCTION public.query_posts_by_ids(
  ids integer[])
    RETURNS SETOF post 
    LANGUAGE 'sql'
AS $BODY$

  SELECT *
  FROM post
  WHERE post.id = ANY(ids)

$BODY$;
```
Lệnh gọi trong Graphql
```json
{
  searchPosts(search: "the", first:3) {
    edges {
      node {
        id
        title
      }
    }
  }
}
```

```sql
SELECT public.query_posts_by_ids(ids => ARRAY[1, 2, 3])
```

Lệnh GraphQL
```
{
  queryPostsByIds(ids: [1, 2, 5]) {
    nodes {
      id
      title
      authorPostsByPostId {
        nodes {
          authorByAuthorId {
            firstName
            lastName
          }
        }
      }
    }
  }
}
```

## Mutation - Sửa đổi dữ liệu

Thêm mới post
```
mutation
{
  createPost(input: {    
    post: {
      title: "Rock rap bậy bạ",
      content: "Nhố nhăng quá trơi"
    }
  }
  ) {
    clientMutationId
  }
  
}
```

Sửa một post
```
mutation {  
  updatePostById(input: {
    id: 5,
    postPatch: {
      title: "Cuốn theo chiều gió",
      content: "Sách hay nên đọc"
    }
  }) {
    clientMutationId
  }
}
```