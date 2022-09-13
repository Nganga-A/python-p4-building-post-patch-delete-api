# Building a POST/PATCH/DELETE API

## Learning Goals

- Build an API to handle POST, PATCH, and DELETE requests.

***

## Key Vocab

- **Application Programming Interface (API)**: a software application that
  allows two or more software applications to communicate with one another.
  Can be standalone or incorporated into a larger product.
- **HTTP Request Method**: assets of HTTP requests that tell the server which
  actions the client is attempting to perform on the located resource.
- **`GET`**: the most common HTTP request method. Signifies that the client is
  attempting to view the located resource.
- **`POST`**: the second most common HTTP request method. Signifies that the
  client is attempting to submit a form to create a new resource.
- **`PATCH`**: an HTTP request method that signifies that the client is attempting
  to update a resource with new information.
- **`DELETE`**: an HTTP request method that signifies that the client is
  attempting to delete a resource.

***

## Introduction

So far, we've seen how to set up an API with Flask to allow frontend
applications to access data from a database in a JSON format. For many
applications, just being able to access/read data isn't enough — what kind of
app would Twitter be if you couldn't write posts? What would Instagram be if you
couldn't like photos? How embarrassing would Facebook be if you couldn't go back
and delete those regrettable high school photos?

All of those applications, and most web apps, can be broadly labeled as CRUD
applications — they allow users to **C**reate, **R**ead, **U**pdate, and
**D**elete information.

We've seen a few ways to Read data in an API. We've also already seen how to
Create/Update/Delete records from a database using SQLAlchemy. All that's
left is to connect what we know from SQLAlchemy with some new techniques for
establishing routes and accessing data in our Flask application.

***

## Setup

We'll continue working on the game review application from the previous lessons.
To get set up, run:

```console
$ pipenv install && pipenv shell
$ cd app
$ flask db upgrade
$ python seed.py
```

You can view the models in the `app/models.py` module, and the migrations in the
`app/migrations/versions` directory. Here's what the relationships will look
like in our ERD:

![Game Reviews ERD](https://curriculum-content.s3.amazonaws.com/phase-3/active-record-associations-many-to-many/games-reviews-users-erd.png)

`app/app.py` has also been configured with `GET` routes for all `Review` and
`User` records.

Now, run the server with Flask and re-explore some of the routes from our `GET`
lesson and the new `/reviews` and `/users` routes:

```console
$ flask run
```

With that set up, let's start working on some CRUD!

***

## Handling DELETE Requests

Let's start with the simplest action: the DELETE request. Imagine we're building
a new feature in our frontend React application. Our users want some way to
delete their reviews, in case they change their minds. In React, our component
for handling this delete action might look something like this:

```js
function ReviewItem({ review, onDeleteReview }) {
  function handleDeleteClick() {
    fetch(`http://localhost:9292/reviews/${review.id}`, {
      method: "DELETE",
    })
      .then((r) => r.json())
      .then((deletedReview) => onDeleteReview(deletedReview));
  }

  return (
    <div>
      <p>Score: {review.score}</p>
      <p>{review.comment}</p>
      <button onClick={handleDeleteClick}>Delete Review</button>
    </div>
  );
}
```

So, it looks like our server needs to handle a few new things:

- Handle requests with the `DELETE` HTTP verb to `/reviews/:id`.
- Find the review to delete using the ID.
- Delete the review from the database.
- Send a response with the deleted review as JSON to confirm that it was deleted
  successfully, so the frontend can show the successful deletion to the user.

Let's take things one step at a time. First, we'll need to handle requests by
adding a new route in the controller. We can write out a route for a DELETE
request just like we would for a GET request, just by changing the method:

```py
# app/app.py

# imports, games, game_by_id, reviews

@app.route('/reviews/<int:id>', methods=['GET', 'DELETE'])
def review_by_id(id):
    review = Review.query.filter_by(id=id).first()

    if request.method == 'GET':
        review_dict = review.to_dict()

        response = make_response(
            jsonify(review_dict),
            200
        )
        response.headers["Content-Type"] = "application/json"

        return response

    elif request.method == 'DELETE':
        db.session.delete(review)
        db.session.commit()

        response_body = {
            "delete_successful": True,
            "message": "Review deleted."    
        }

        response = make_response(
            jsonify(response_body),
            200
        )

        return response

    else:
        response_body = {
            "delete_successful": False,
            "message": "HTTP method not supported."}
        
        response = make_response(
            jsonify(response_body),
            405
        )

        return response
```

Let's review the new content:

- The `@app.route` decorator accepts `methods` as a default argument. This is
  simply a list of accepted methods as strings. By default, this list only
  contains `'GET'`.
- The request contxt allows us to access the HTTP method used by the request and
  control flow from there. If the request's method is `GET`, we perform the same
  actions that we did in the `/games/<int:id>` route. If the method is `DELETE`,
  we delete the resource.
- Just in case, we added a third block for unsupported methods. These receive a
  405 response code.

Now, the question on everyone's minds: how do we actually send a `DELETE`
request?

### Sending Requests with HTTP Methods Other than `GET`

We all know from experience that we can delete resources on the internet:
emails, Instagram posts with unmade beds in the background, etc. Many of these
can be performed from the browser, but we don't support them from our API. _Why
is that?_

Browsers are actually very flexible on this front- it's HTML that isn't. The
W3C specifications for HTML say that it should only support `GET` and `POST`.
This means that any requests sent to the API through the browser with
unsupported methods will have these methods cast to `GET`...which will then be
used to update and delete resources. (We don't want that!)

We can use Python scripts and applications with libraries like `requests` to
test our `DELETE` resource, but it is often easier to just use Postman.

> **NOTE: Requests to `localhost` or `127.0.0.1` can only be sent using the
> Postman desktop client. This is unsupported in the cloud.**

Sending a `GET` request (the default method) for the first review will return
something similar to the following:

![GET request Postman for reviews/1 on localhost server with JSON response](
https://curriculum-content.s3.amazonaws.com/python/building-post-patch-delete-api-postman-1.png
)

I know what you're thinking: how could we possibly delete this illuminating
review of "All lawyer as teacher world any?" How will anyone know that
"Republican help young large treatment note"?

Alas, we need to make sure our API works. Change the request method to `DELETE`
with the dropdown menu to the left of the search bar and submit a new request to
this resource.

![DELETE request Postman for reviews/1 on localhost server with JSON response](
https://curriculum-content.s3.amazonaws.com/python/building-post-patch-delete-api-postman-2.png
)

> **NOTE: You may notice that the port accessed here has switched from 5000 to
> 5001. Some firewalls block certain ports- sometimes they only start doing so
> after you've been using it for a while! Be prepared to switch ports with
> `flask run -p <port_number>` if you start to see 403s in Postman.**

Navigate back to the same resource with `GET` and you should see a `500
Internal Server Error`. There's no resource here anymore! This is obviously not
the most helpful message, but it can be handled with control flow just like any
other error. A simple solution will be included in the solution code at the end
of this lesson.

***

## Handling POST Requests

For our next feature, let's give our users the ability to **Create** new reviews.
From the frontend, here's how our React component might look:

```js
function ReviewForm({ userId, gameId, onAddReview }) {
  const [comment, setComment] = useState("");
  const [score, setScore] = useState("0");

  function handleSubmit(e) {
    e.preventDefault();
    fetch("http://localhost:9292/reviews", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        comment: comment,
        score: score,
        user_id: userId,
        game_id: gameId,
      }),
    })
      .then((r) => r.json())
      .then((newReview) => onAddReview(newReview));
  }

  return <form onSubmit={handleSubmit}>{/* controlled form code here*/}</form>;
}
```

This request is a bit trickier than the last: in order to create a review in the
database, we need some way of getting all the data that the user entered into
the form. From the code above, you can see that we'll have access to that data
in the **body** of the request, as a JSON-formatted string. So in terms of the
steps for our server, we need to:

- Handle requests with the `POST` HTTP verb to `/reviews`
- Access the data in the body of the request
- Use that data to create a new review in the database
- Send a response with newly created review as JSON

Let's start with the easy part. We can create a new route like so:

***

## Conclusion

.

***

## Solution Code

You're at the point now where you can create a JSON API that handles all four
CRUD actions: Create, Read, Update, and Delete. With just these four actions,
you can build an API for almost any application you can think of!

***

## Resources

- [Flask - Pallets](https://flask.palletsprojects.com/en/2.2.x/)
- [POST - Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST)
- [PATCH - Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/PATCH)
- [DELETE - Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/DELETE)
- [flask.json.jsonify Example Code - Full Stack Python](https://www.fullstackpython.com/flask-json-jsonify-examples.html)
- [SQLAlchemy-serializer - PyPI](https://pypi.org/project/SQLAlchemy-serializer/)