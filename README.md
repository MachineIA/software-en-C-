using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace TempoControlApp
{
    
    public class Empleado
    {
        public int Id { get; set; }
        public string NombreCompleto { get; set; } = "";
        public string Departamento { get; set; } = "";
        public string Posicion { get; set; } = "";
        public bool Activo { get; set; } = true;
        public DateTime FechaCreacion { get; set; } = DateTime.UtcNow;
    }

    public enum TipoFichaje
    {
        Entrada = 0,
        Salida = 1
    }

    public class RegistroFichaje
    {
        public int Id { get; set; }
        public int EmpleadoId { get; set; }
        public DateTime FechaHora { get; set; }
        public TipoFichaje Tipo { get; set; }
    }

    public class FakeDbContext
    {
        public List<Empleado> Empleados { get; set; } = new List<Empleado>();
        public List<RegistroFichaje> RegistrosFichaje { get; set; } = new List<RegistroFichaje>();

        private int _empleadoId = 1;
        private int _registroId = 1;

        public int NextEmpleadoId()
        {
            int id = _empleadoId;
            _empleadoId++;
            return id;
        }

        public int NextRegistroId()
        {
            int id = _registroId;
            _registroId++;
            return id;
        }

        public Task<int> SaveChangesAsync()
        {
            return Task.FromResult(0);
        }

        public void Dispose()
        {
        }
    }

    public interface IRepository<T> where T : class
    {
        Task<T?> GetByIdAsync(int id);
        Task<List<T>> GetAllAsync();
        Task AddAsync(T entity);
        void Update(T entity);
        void Remove(T entity);
        Task<List<T>> FindAsync(Func<T, bool> predicate);
    }

    public class Repository<T> : IRepository<T> where T : class
    {
        protected readonly FakeDbContext _context;

        public Repository(FakeDbContext context)
        {
            _context = context;
        }

        public virtual Task AddAsync(T entity)
        {
            if (typeof(T) == typeof(Empleado))
            {
                Empleado e = (Empleado)(object)entity;
                if (e.Id == 0) e.Id = _context.NextEmpleadoId();
                _context.Empleados.Add(e);
            }
            else if (typeof(T) == typeof(RegistroFichaje))
            {
                RegistroFichaje r = (RegistroFichaje)(object)entity;
                if (r.Id == 0) r.Id = _context.NextRegistroId();
                _context.RegistrosFichaje.Add(r);
            }
            return Task.CompletedTask;
        }

        public virtual Task<List<T>> FindAsync(Func<T, bool> predicate)
        {
            List<T> list = GetList().Where(predicate).ToList();
            return Task.FromResult(list);
        }

        public virtual Task<List<T>> GetAllAsync()
        {
            List<T> list = GetList().ToList();
            return Task.FromResult(list);
        }

        public virtual Task<T?> GetByIdAsync(int id)
        {
            T? entity = default(T);
            if (typeof(T) == typeof(Empleado))
            {
                entity = (T?)(object?)_context.Empleados.FirstOrDefault(e => e.Id == id);
            }
            else if (typeof(T) == typeof(RegistroFichaje))
            {
                entity = (T?)(object?)_context.RegistrosFichaje.FirstOrDefault(r => r.Id == id);
            }
            return Task.FromResult(entity);
        }

        public virtual void Remove(T entity)
        {
            if (typeof(T) == typeof(Empleado))
            {
                _context.Empleados.Remove((Empleado)(object)entity);
            }
            else if (typeof(T) == typeof(RegistroFichaje))
            {
                _context.RegistrosFichaje.Remove((RegistroFichaje)(object)entity);
            }
        }

        public virtual void Update(T entity)
        {
        }

        private IEnumerable<T> GetList()
        {
            if (typeof(T) == typeof(Empleado))
            {
                return _context.Empleados.Cast<T>();
            }
            else if (typeof(T) == typeof(RegistroFichaje))
            {
                return _context.RegistrosFichaje.Cast<T>();
            }
            return new List<T>();
        }
    }


    public interface IEmpleadoRepository : IRepository<Empleado>
    {
        Task<List<Empleado>> GetActivosAsync();
        Task DeactivateAsync(int id);
    }

    public class EmpleadoRepository : Repository<Empleado>, IEmpleadoRepository
    {
        public EmpleadoRepository(FakeDbContext context) : base(context) { }

        public Task<List<Empleado>> GetActivosAsync()
        {
            List<Empleado> list = _context.Empleados.Where(e => e.Activo).ToList();
            return Task.FromResult(list);
        }

        public Task DeactivateAsync(int id)
        {
            Empleado? emp = _context.Empleados.FirstOrDefault(e => e.Id == id);
            if (emp != null)
            {
                emp.Activo = false;
            }
            return Task.CompletedTask;
        }
    }

 

    public interface IFichajeRepository : IRepository<RegistroFichaje>
    {
        Task<List<RegistroFichaje>> GetByEmpleadoAndMonthAsync(int empleadoId, int month, int year);
        Task<List<RegistroFichaje>> GetByMonthAsync(int month, int year);
    }

    public class FichajeRepository : Repository<RegistroFichaje>, IFichajeRepository
    {
        public FichajeRepository(FakeDbContext context) : base(context) { }

        public Task<List<RegistroFichaje>> GetByEmpleadoAndMonthAsync(int empleadoId, int month, int year)
        {
            List<RegistroFichaje> list = _context.RegistrosFichaje
                .Where(r => r.EmpleadoId == empleadoId &&
                            r.FechaHora.Year == year &&
                            r.FechaHora.Month == month)
                .OrderBy(r => r.FechaHora)
                .ToList();
            return Task.FromResult(list);
        }

        public Task<List<RegistroFichaje>> GetByMonthAsync(int month, int year)
        {
            List<RegistroFichaje> list = _context.RegistrosFichaje
                .Where(r => r.FechaHora.Year == year &&
                            r.FechaHora.Month == month)
                .OrderBy(r => r.EmpleadoId)
                .ThenBy(r => r.FechaHora)
                .ToList();
            return Task.FromResult(list);
        }
    }

   

    public interface IUnitOfWork : IDisposable
    {
        IEmpleadoRepository Empleados { get; }
        IFichajeRepository Fichajes { get; }
        Task<int> CompleteAsync();
    }

    public class UnitOfWork : IUnitOfWork
    {
        private readonly FakeDbContext _context;
        public IEmpleadoRepository Empleados { get; private set; }
        public IFichajeRepository Fichajes { get; private set; }

        public UnitOfWork(FakeDbContext context)
        {
            _context = context;
            Empleados = new EmpleadoRepository(_context);
            Fichajes = new FichajeRepository(_context);
        }

        public Task<int> CompleteAsync()
        {
            return _context.SaveChangesAsync();
        }

        public void Dispose()
        {
            _context.Dispose();
        }
    }

   

    public class EmpleadoService
    {
        private readonly IUnitOfWork _uow;
        public EmpleadoService(IUnitOfWork uow)
        {
            _uow = uow;
        }

        public Task<List<Empleado>> ObtenerTodosAsync()
        {
            return _uow.Empleados.GetAllAsync();
        }

        public Task<Empleado?> ObtenerPorIdAsync(int id)
        {
            return _uow.Empleados.GetByIdAsync(id);
        }

        public async Task CrearEmpleadoAsync(Empleado e)
        {
            await _uow.Empleados.AddAsync(e);
            await _uow.CompleteAsync();
        }

        public async Task ActualizarEmpleadoAsync(Empleado e)
        {
            _uow.Empleados.Update(e);
            await _uow.CompleteAsync();
        }

        public async Task DesactivarEmpleadoAsync(int id)
        {
            await _uow.Empleados.DeactivateAsync(id);
            await _uow.CompleteAsync();
        }
    }

    public class ReporteEmpleadoMes
    {
        public int EmpleadoId { get; set; }
        public string Nombre { get; set; } = "";
        public int DiasTrabajados { get; set; }
        public double HorasTotales { get; set; }
    }

    public class FichajeService
    {
        private readonly IUnitOfWork _uow;
        public FichajeService(IUnitOfWork uow)
        {
            _uow = uow;
        }

        public async Task RegistrarFichajeAsync(int empleadoId, TipoFichaje tipo, DateTime fechaHoraUtc)
        {
            Empleado? empleado = await _uow.Empleados.GetByIdAsync(empleadoId);
            if (empleado == null || !empleado.Activo)
            {
                throw new InvalidOperationException("Empleado no existe o no está activo.");
            }

            RegistroFichaje registro = new RegistroFichaje
            {
                EmpleadoId = empleadoId,
                Tipo = tipo,
                FechaHora = fechaHoraUtc
            };

            await _uow.Fichajes.AddAsync(registro);
            await _uow.CompleteAsync();
        }

        public async Task<List<ReporteEmpleadoMes>> GenerarReporteMensualAsync(int month, int year)
        {
            List<Empleado> empleados = await _uow.Empleados.GetAllAsync();
            List<RegistroFichaje> registros = await _uow.Fichajes.GetByMonthAsync(month, year);

            List<ReporteEmpleadoMes> result = new List<ReporteEmpleadoMes>();

            foreach (Empleado emp in empleados)
            {
                List<RegistroFichaje> regs = registros
                    .Where(r => r.EmpleadoId == emp.Id)
                    .OrderBy(r => r.FechaHora)
                    .ToList();

                var porDia = regs.GroupBy(r => r.FechaHora.Date);

                double totalHoras = 0;
                int diasTrabajados = 0;

                foreach (var grupo in porDia)
                {
                    List<RegistroFichaje> listaDia = grupo.ToList();

                    DateTime? entrada = null;
                    double horasDia = 0;

                    for (int i = 0; i < listaDia.Count; i++)
                    {
                        RegistroFichaje r = listaDia[i];
                        if (r.Tipo == TipoFichaje.Entrada)
                        {
                            entrada = r.FechaHora;
                        }
                        else if (r.Tipo == TipoFichaje.Salida && entrada.HasValue)
                        {
                            DateTime salida = r.FechaHora;
                            TimeSpan diff = salida - entrada.Value;
                            horasDia += diff.TotalHours;
                            entrada = null;
                        }
                    }

                    if (horasDia > 0)
                    {
                        totalHoras += horasDia;
                        diasTrabajados++;
                    }
                }

                ReporteEmpleadoMes rep = new ReporteEmpleadoMes
                {
                    EmpleadoId = emp.Id,
                    Nombre = emp.NombreCompleto,
                    DiasTrabajados = diasTrabajados,
                    HorasTotales = Math.Round(totalHoras, 2)
                };

                result.Add(rep);
            }

            return result;
        }
    }

    

    public class Program
    {
        public static async Task Main(string[] args)
        {
            FakeDbContext db = new FakeDbContext();
            IUnitOfWork uow = new UnitOfWork(db);
            EmpleadoService empleadoService = new EmpleadoService(uow);
            FichajeService fichajeService = new FichajeService(uow);

            while (true)
            {
                Console.WriteLine("=== TempoControl (Memoria) ===");
                Console.WriteLine("1) Gestionar Empleados (CRUD)");
                Console.WriteLine("2) Registrar Entrada");
                Console.WriteLine("3) Registrar Salida");
                Console.WriteLine("4) Generar Reporte Mensual");
                Console.WriteLine("0) Salir");
                Console.Write("Seleccione: ");
                string? opt = Console.ReadLine();

                if (opt == "0")
                {
                    break;
                }

                try
                {
                    if (opt == "1")
                    {
                        await MenuEmpleados(empleadoService);
                    }
                    else if (opt == "2")
                    {
                        await RegistrarFichaje(fichajeService, TipoFichaje.Entrada);
                    }
                    else if (opt == "3")
                    {
                        await RegistrarFichaje(fichajeService, TipoFichaje.Salida);
                    }
                    else if (opt == "4")
                    {
                        await GenerarReporte(fichajeService);
                    }
                    else
                    {
                        Console.WriteLine("Opción inválida.");
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine("Error: " + ex.Message);
                }

                Console.WriteLine();
            }
        }

        private static async Task MenuEmpleados(EmpleadoService svc)
        {
            while (true)
            {
                Console.WriteLine("---- Empleados ----");
                Console.WriteLine("1) Listar todos");
                Console.WriteLine("2) Crear nuevo");
                Console.WriteLine("3) Actualizar");
                Console.WriteLine("4) Desactivar");
                Console.WriteLine("0) Volver");
                Console.Write("Opción: ");
                string? op = Console.ReadLine();
                if (op == "0")
                {
                    break;
                }

                if (op == "1")
                {
                    List<Empleado> all = await svc.ObtenerTodosAsync();
                    Console.WriteLine("Id | Nombre | Departamento | Posición | Activo");
                    foreach (Empleado e in all)
                    {
                        Console.WriteLine(e.Id + " | " + e.NombreCompleto + " | " + e.Departamento + " | " + e.Posicion + " | " + e.Activo);
                    }
                }
                else if (op == "2")
                {
                    Console.Write("Nombre completo: ");
                    string nombre = Console.ReadLine() ?? "";
                    Console.Write("Departamento: ");
                    string dept = Console.ReadLine() ?? "";
                    Console.Write("Posición: ");
                    string pos = Console.ReadLine() ?? "";
                    Empleado emp = new Empleado { NombreCompleto = nombre, Departamento = dept, Posicion = pos };
                    await svc.CrearEmpleadoAsync(emp);
                    Console.WriteLine("Empleado creado.");
                }
                else if (op == "3")
                {
                    Console.Write("Id del empleado a actualizar: ");
                    int uid;
                    if (int.TryParse(Console.ReadLine(), out uid))
                    {
                        Empleado? empu = await svc.ObtenerPorIdAsync(uid);
                        if (empu == null)
                        {
                            Console.WriteLine("No existe.");
                        }
                        else
                        {
                            Console.Write("Nombre (" + empu.NombreCompleto + "): ");
                            string? nn = Console.ReadLine();
                            Console.Write("Departamento (" + empu.Departamento + "): ");
                            string? nd = Console.ReadLine();
                            Console.Write("Posición (" + empu.Posicion + "): ");
                            string? np = Console.ReadLine();
                            if (!string.IsNullOrWhiteSpace(nn)) empu.NombreCompleto = nn;
                            if (!string.IsNullOrWhiteSpace(nd)) empu.Departamento = nd;
                            if (!string.IsNullOrWhiteSpace(np)) empu.Posicion = np;
                            await svc.ActualizarEmpleadoAsync(empu);
                            Console.WriteLine("Actualizado.");
                        }
                    }
                    else
                    {
                        Console.WriteLine("Id inválido.");
                    }
                }
                else if (op == "4")
                {
                    Console.Write("Id a desactivar: ");
                    int idd;
                    if (int.TryParse(Console.ReadLine(), out idd))
                    {
                        await svc.DesactivarEmpleadoAsync(idd);
                        Console.WriteLine("Desactivado.");
                    }
                    else
                    {
                        Console.WriteLine("Id inválido.");
                    }
                }
                else
                {
                    Console.WriteLine("Opción inválida.");
                }
            }
        }

        private static async Task RegistrarFichaje(FichajeService svc, TipoFichaje tipo)
        {
            Console.Write("Id del empleado: ");
            int id;
            if (!int.TryParse(Console.ReadLine(), out id))
            {
                Console.WriteLine("Id inválido.");
                return;
            }
            DateTime fecha = DateTime.UtcNow;
            await svc.RegistrarFichajeAsync(id, tipo, fecha);
            Console.WriteLine(tipo + " registrada para empleado " + id + " a " + fecha.ToString("yyyy-MM-dd HH:mm:ss") + " (UTC).");
        }

        private static async Task GenerarReporte(FichajeService svc)
        {
            Console.Write("Mes (1-12): ");
            int mes;
            if (!int.TryParse(Console.ReadLine(), out mes))
            {
                Console.WriteLine("Mes inválido.");
                return;
            }
            Console.Write("Año (ej. 2025): ");
            int anio;
            if (!int.TryParse(Console.ReadLine(), out anio))
            {
                Console.WriteLine("Año inválido.");
                return;
            }

            List<ReporteEmpleadoMes> reporte = await svc.GenerarReporteMensualAsync(mes, anio);
            Console.WriteLine("Reporte " + mes + "/" + anio);
            Console.WriteLine("Empleado | Días trabajados | Horas totales");
            foreach (ReporteEmpleadoMes r in reporte)
            {
                Console.WriteLine(r.Nombre + " | " + r.DiasTrabajados + " | " + r.HorasTotales);
            }
        }
    }
}
