# Glossary

> 직관적으로 정리

1. tensor(정보)
  - Tensor는 weight matrix를 통과(계산)해도 이전의 정보는 보존된다
2. weighted sum(softmax weight)
  - 다른 정보를 가진 tensor를 weighted sum을 하면, 당연히 weight이 큰 쪽의 정보를 더 많이 갖고있다
  - 정보의 짬뽕이며, weight가 큰 쪽의 정보가 더 많이 들어간다
  - 이 형태를 만들기위해, exponential 후 softmax로 비중을 구한다
3. inner product(내적)
  - 벡터 두개로 내적한다(같은 차원 숫자끼리 곱해서 더함)
  - 서로 비슷한 정보들의 내적은 상대적으로 값이 크다(10)
  - 서로 상관 없는 정보들의 내적은 값이 상대적으로 작다(-10)