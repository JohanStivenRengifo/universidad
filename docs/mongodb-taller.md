# mongodb taller

Mostrar los usuarios que han realizado mas prestamos dentro del rango de una semana

{% code expandable="true" %}
```javascript
db.prestamos.aggregate([
  {
    $match: {
      fecha_prestamo: {
        $gte: ISODate("2026-09-28"),
        $lt: ISODate("2026-10-05")
      }
    }
  },
  {
    $group: {
      _id: "$usuario_id",
      cantidad_prestamos: {
        $sum: 1
      }
    }
  },
  {
    $sort: {
      cantidad_prestamos: -1
    }
  },
  {
    $lookup: {
      from: "usuarios",
      localField: "_id",
      foreignField: "_id",
      as: "usuario"
    }
  },
  {
    $unwind: "$usuario"
  },
  {
    $project: {
      _id: 0,
      usuario: "$usuario.nombre",
      cantidad_prestamos: 1
    }
  }
]);
```
{% endcode %}

resultado

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Mostrar cuantos libros se encuentran disponibles
